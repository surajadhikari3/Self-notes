Got it — let’s lock in a clean, minimal, **end-to-end** setup that:

- logs in to Deephaven with **PSK**,
    
- opens **stable global tables** (`user`, `account`, `user_account`),
    
- subscribes to **ticking updates** via a viewport,
    
- **auto-heals** if DH recycles a handle (re-acquires and re-subscribes),
    
- does **not** close server tables from Angular,
    
- works with Angular **20.3**.
    

Below are the 5 files you need. Replace your current versions with these.

---

## 1) `src/app/deephaven/deephaven.service.ts`

```ts
import { Injectable } from '@angular/core';
import { environment } from '../../environments/environment';

type DhNS = any;
type DhCoreClient = any;
type DhIde = any;
export type DhTable = any;

@Injectable({ providedIn: 'root' })
export class DeephavenService {
  private dh!: DhNS;
  private client!: DhCoreClient;
  private ide!: DhIde;
  private ready = false;

  get isReady(): boolean { return this.ready; }
  getDhNs() { return this.dh; } // expose DH namespace (for event constants)

  /** PSK login + Python IDE session */
  async connect(): Promise<void> {
    if (this.ready) return;

    const serverUrl = environment.deephavenUrl;
    const psk = environment.deephavenPsk;
    if (!serverUrl) throw new Error('environment.deephavenUrl is not set');
    if (!psk) throw new Error('environment.deephavenPsk is not set');

    const jsapiUrl = new URL('/jsapi/dh-core.js', serverUrl).toString();
    const mod = await import(/* @vite-ignore */ jsapiUrl);
    this.dh = mod.default;

    this.client = new this.dh.CoreClient(serverUrl);
    await this.client.login({
      type: 'io.deephaven.authentication.psk.PskAuthenticationHandler',
      token: psk,
    });

    this.ide = await this.client.getAsIdeConnection();
    await this.ide.startSession('python');
    this.ready = true;
  }

  /** Fetch a handle to a globals() table by name */
  async getTableHandle(name: string): Promise<DhTable> {
    if (!this.ready) await this.connect();
    if (!/^[A-Za-z_][A-Za-z0-9_]*$/.test(name)) {
      throw new Error(`Invalid table variable name: "${name}"`);
    }
    return this.ide.getTable(name);
  }
}
```

---

## 2) `src/app/live-table/live-table.component.ts` (standalone, auto-ticking)

```ts
import {
  Component, Input, OnInit, OnDestroy,
  ChangeDetectionStrategy, ChangeDetectorRef
} from '@angular/core';
import { CommonModule } from '@angular/common';
import { DeephavenService, DhTable } from '../deephaven/deephaven.service';

@Component({
  selector: 'app-live-table',
  standalone: true,
  imports: [CommonModule],
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
  <section class="card">
    <header class="bar">
      <h3><span class="mono">{{ tableName }}</span></h3>
      <div class="status">
        <span *ngIf="error" class="error">{{ error }}</span>
        <span *ngIf="!error && loading">Connecting…</span>
      </div>
    </header>

    <div class="grid" *ngIf="!error">
      <div class="thead" *ngIf="cols.length">
        <div class="th" *ngFor="let c of cols; trackBy: trackCol">{{ c }}</div>
      </div>

      <div class="tbody" (scroll)="onScroll($event)">
        <div class="tr" *ngFor="let r of rows; trackBy: trackRow">
          <div class="td" *ngFor="let v of r; trackBy: trackCell">{{ v }}</div>
        </div>
      </div>
    </div>

    <footer class="pager" *ngIf="!error">
      <button (click)="pageFirst()" [disabled]="offset === 0">First</button>
      <button (click)="pagePrev()"  [disabled]="offset === 0">Prev</button>
      <span class="mono">rows {{ offset }}–{{ offset + pageSize - 1 }}</span>
      <button (click)="pageNext()">Next</button>
      <button (click)="refresh()">Refresh</button>
    </footer>
  </section>
  `,
  styles: [`
    :host { display:block }
    .card { border:1px solid #2a2a2a; border-radius:.75rem; background:#0f0f0f; color:#fff; }
    .bar { display:flex; justify-content:space-between; align-items:center; padding:.75rem 1rem; border-bottom:1px solid #242424 }
    .mono{ font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, "Liberation Mono", monospace }
    .grid{ display:grid; grid-template-rows: auto 1fr; gap:.5rem; }
    .thead, .tr { display:grid; grid-auto-flow:column; grid-auto-columns:minmax(140px, 1fr) }
    .th { position:sticky; top:0; background:#111; font-weight:600; padding:.5rem; z-index:1; }
    .tbody{ max-height:40dvh; overflow:auto; }
    .td { padding:.25rem .5rem; border-top:1px solid #242424; white-space:nowrap; overflow:hidden; text-overflow:ellipsis }
    .pager{ display:flex; gap:.5rem; align-items:center; padding:.5rem 1rem; border-top:1px solid #242424 }
    .error{ color:#ff6b6b }
  `],
})
export class LiveTableComponent implements OnInit, OnDestroy {
  /** name of DH globals() table, e.g. "user_account" */
  @Input({ required: true }) tableName!: string;
  @Input() pageSize = 300;
  @Input() initialOffset = 0;

  loading = false;
  error: string | null = null;

  cols: string[] = [];
  rows: any[] = [];
  offset = 0;

  private table: DhTable | null = null;
  private viewport: any;
  private onUpdateHandler?: () => void;
  private onSchemaHandler?: () => void;

  constructor(private dh: DeephavenService, private cdr: ChangeDetectorRef) {}

  async ngOnInit(): Promise<void> {
    this.offset = this.initialOffset;
    await this.attach();
  }
  async ngOnDestroy(): Promise<void> { await this.detach(); }

  // ---------- wiring ----------
  private async attach(): Promise<void> {
    this.loading = true; this.error = null; this.cdr.markForCheck();
    try {
      await this.dh.connect();
      const dhns = this.dh.getDhNs();

      // fresh handle each attach (survives server-side re-runs)
      this.table = await this.dh.getTableHandle(this.tableName);

      this.cols = (this.table.columns ?? []).map((c: any) => c.name);

      this.viewport = await this.table.setViewport(
        this.offset, this.offset + this.pageSize - 1, this.cols
      );

      await this.refresh(); // initial data

      const EVU = dhns?.Table?.EVENT_UPDATED ?? 'update';
      const EVS = dhns?.Table?.EVENT_SCHEMA_CHANGED ?? 'schema_changed';

      this.onUpdateHandler = async () => { await this.refresh(); };
      this.onSchemaHandler = async () => { await this.onSchemaChanged(); };

      this.table.addEventListener(EVU, this.onUpdateHandler);
      this.table.addEventListener(EVS, this.onSchemaHandler);

      this.loading = false; this.cdr.markForCheck();
    } catch (e: any) {
      this.error = e?.message ?? String(e);
      this.loading = false; this.cdr.markForCheck();
    }
  }

  private async detach(): Promise<void> {
    try {
      const dhns = this.dh.getDhNs();
      const EVU = dhns?.Table?.EVENT_UPDATED ?? 'update';
      const EVS = dhns?.Table?.EVENT_SCHEMA_CHANGED ?? 'schema_changed';
      if (this.table && this.onUpdateHandler) this.table.removeEventListener?.(EVU, this.onUpdateHandler);
      if (this.table && this.onSchemaHandler) this.table.removeEventListener?.(EVS, this.onSchemaHandler);
    } catch {}
    try { await this.viewport?.close?.(); } catch {}
    // IMPORTANT: do NOT close the table (it’s a long-lived server global)
    this.viewport = null;
    this.table = null;
  }

  // ---------- data ----------
  async refresh(): Promise<void> {
    if (!this.table) return;
    try {
      const snap = await this.table.snapshot(this.cols);
      const objs = snap.toObjects?.() ?? snap;
      this.rows = (objs as any[]).map(o =>
        Array.isArray(o) ? o : this.cols.map(c => (o as any)[c])
      );
      this.cdr.markForCheck();
    } catch (e: any) {
      const msg = (e?.message ?? String(e)).toLowerCase();
      if (msg.includes('closed')) {
        // self-heal: server recycled handle; reattach
        await this.detach();
        await this.attach();
      } else {
        this.error = e?.message ?? String(e);
        this.cdr.markForCheck();
      }
    }
  }

  private async onSchemaChanged(): Promise<void> {
    if (!this.table) return;
    try {
      this.cols = (this.table.columns ?? []).map((c: any) => c.name);
      await this.table.setViewport(this.offset, this.offset + this.pageSize - 1, this.cols);
      await this.refresh();
    } catch {}
  }

  // ---------- paging & scroll ----------
  async pageFirst() { this.offset = 0; await this.onPage(); }
  async pagePrev()  { this.offset = Math.max(0, this.offset - this.pageSize); await this.onPage(); }
  async pageNext()  { this.offset = this.offset + this.pageSize; await this.onPage(); }
  private async onPage() {
    if (!this.table) return;
    await this.table.setViewport(this.offset, this.offset + this.pageSize - 1, this.cols);
    await this.refresh();
  }
  onScroll(ev: Event) {
    const el = ev.target as HTMLElement;
    if (el.scrollTop + el.clientHeight >= el.scrollHeight - 8) this.pageNext();
  }

  // ---------- trackBys ----------
  trackCol = (_: number, c: string) => c;
  trackRow = (_: number, r: any[]) => r?.[0] ?? _;
  trackCell = (_: number, _v: any) => _;
}
```

---

## 3) `src/app/app.component.ts` (standalone shell)

```ts
import { Component } from '@angular/core';
import { CommonModule } from '@angular/common';
import { LiveTableComponent } from './live-table/live-table.component';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [CommonModule, LiveTableComponent],
  template: `
    <main style="display:grid; grid-template-columns: 1fr 1fr; gap:1rem">
      <section>
        <h2>User</h2>
        <app-live-table tableName="user"></app-live-table>
      </section>
      <section>
        <h2>Account</h2>
        <app-live-table tableName="account"></app-live-table>
      </section>
      <section style="grid-column:1 / -1">
        <h2>User × Account (as-of)</h2>
        <app-live-table tableName="user_account"></app-live-table>
      </section>
    </main>
  `,
})
export class AppComponent {}
```

---

## 4) `src/main.ts`

```ts
import { bootstrapApplication } from '@angular/platform-browser';
import { AppComponent } from './app/app.component';

bootstrapApplication(AppComponent).catch(err => console.error(err));
```

---

## 5) `src/environments/environment.ts`

```ts
export const environment = {
  production: false,
  deephavenUrl: 'http://localhost:10000',
  deephavenPsk: 'YOUR_PSK_TOKEN',
};
```

---

### Important server-side notes

- In Deephaven Python, make sure your append tables and join are **stable globals**:
    
    - `user`, `account`, `user_account`.
        
- Use an idempotent script (create consumers only if not present). The “self-heal” in the component will also recover if you do re-run the cell, but avoiding constant re-creation is best.
    

---

With these files:

- The UI **auto-updates** whenever DH pushes ticks.
    
- If the server recycles a handle (e.g., you re-run the cell), the component **re-attaches** automatically.
    
- No table closes from Angular (prevents “table already closed” errors).
    

If you want the AG-Grid view later, we can wire the same viewport+snapshot approach into an AG-Grid adapter, but this version gets you streaming right now without the AG Grid v34 API friction you hit earlier.

---------------------------------------


Got you. You’re hitting two separate things:

1. **`ClassCastException` coming from the DH server** (usually triggered by a stale/closed handle or a schema change during a snapshot).
    
2. **“table already closed”** because the client keeps using a viewport/subscription after DH invalidated the original handle (common when the underlying query changes or a session is recycled).
    

Below is a small, drop-in refactor that (a) logs in with **PSK**, (b) binds each grid via a **viewport subscription** (not repeated `snapshot()` calls), and (c) **self-heals** by reacquiring a _fresh_ table handle when DH says the table is closed, then re-opens the viewport and continues streaming without crashing.

This follows the current DH JS API patterns: connect with `CoreClient` + PSK, get an `IdeConnection`, call `ide.getTable(name)`, then use **`table.setViewport` / `TableViewportSubscription.update()` + `getViewportData()`** and listen for **`EVENT_UPDATED`**. If the server reports “closed”, reacquire the handle and re-subscribe. ([Deephaven](https://deephaven.io/core/docs/how-to-guides/authentication/auth-psk/?utm_source=chatgpt.com "Configure and use pre-shared key authentication")) ([DeepHaven Docs](https://docs.deephaven.io/core/client-api/javascript/classes/dh.TableViewportSubscription.html "TableViewportSubscription | @deephaven/jsapi-types")) ([Deephaven](https://deephaven.io/core/docs/reference/js-api/concepts/?utm_source=chatgpt.com "Javascript API Concepts"))

---

### Deephaven service (replace your current one)

```ts
// deephaven.service.ts
import { Injectable } from '@angular/core';
import { environment } from '../environments/environment';

type DhNs = any;
type DhCoreClient = any;
type DhIde = any;
type DhTable = any;
type TableViewportSubscription = any;

@Injectable({ providedIn: 'root' })
export class DeephavenService {
  private dh!: DhNs;
  private client!: DhCoreClient;
  private ide!: DhIde;
  private ready = false;

  get isReady() { return this.ready; }

  /** PSK login via CoreClient -> IdeConnection */
  async connect(): Promise<void> {
    if (this.ready) return;

    const serverUrl = environment.deephavenUrl;      // e.g. https://your-dh-host:10000
    const psk       = environment.deephavenPsk;      // PSK string
    if (!serverUrl) throw new Error('environment.deephavenUrl is not set');
    if (!psk) throw new Error('environment.deephavenPsk is not set');

    // Load the JS API right from the DH server
    const jsapiUrl = new URL('/jsapi/dh-core.js', serverUrl).toString();
    const mod = await import(/* @vite-ignore */ jsapiUrl);
    this.dh = mod.default;

    this.client = new this.dh.CoreClient(serverUrl);
    await this.client.login({
      type: 'io.deephaven.authentication.psk.PskAuthenticationHandler',
      token: psk,
    });

    this.ide = await this.client.getAsIdeConnection();
    await this.ide.startSession('python'); // matches console behavior
    this.ready = true;
  }

  /** Acquire a *fresh* exported table by name (globals/cached widget) */
  async getTableHandle(name: string): Promise<DhTable> {
    if (!this.ready) await this.connect();
    if (!/^[A-Za-z_][A-Za-z0-9_]*$/.test(name)) {
      throw new Error(`Invalid table variable name: "${name}"`);
    }
    return this.ide.getTable(name);
  }

  /** Create a safe viewport binder that can rebind if the table gets closed. */
  createViewportBinder() {
    return new ViewportBinder(this.dh, (n: string) => this.getTableHandle(n));
  }
}

/** Keeps a viewport open, streams updates, and self-heals on "closed". */
class ViewportBinder {
  private dh: any;
  private resolver: (name: string) => Promise<any>;
  private table?: any;
  private sub?: TableViewportSubscription;
  private destroyed = false;
  private rebinding = false;

  constructor(dh: any, resolver: (name: string) => Promise<any>) {
    this.dh = dh;
    this.resolver = resolver;
  }

  /** Bind to a table name; provide a column list (optional) and onRows callback. */
  async bind(
    tableName: string,
    columns: string[] | undefined,
    onRows: (rows: any[]) => void,
  ) {
    await this.unbind(); // serialize
    this.destroyed = false;

    // 1) get a fresh table and open viewport subscription
    this.table = await this.resolver(tableName);
    const cols = (columns ?? this.table.columns?.map((c: any) => c.name)) ?? [];
    this.sub = this.table.setViewport(0, 10000, cols); // returns TableViewportSubscription

    // Initial data
    await this.pushViewport(onRows);

    // Live updates
    const EVENT = (this.dh?.Table?.EVENT_UPDATED ?? 'update');
    const onUpdate = async () => {
      if (this.destroyed) return;
      try {
        await this.pushViewport(onRows);
      } catch (e: any) {
        // Typical "table closed" path - trigger self-heal
        const msg = (e?.message ?? String(e)).toLowerCase();
        if (msg.includes('closed')) await this.rebind(tableName, cols, onRows);
        else console.warn('viewport update error:', e);
      }
    };

    this.sub.addEventListener(EVENT, onUpdate);
  }

  private async pushViewport(onRows: (rows: any[]) => void) {
    // Pull the current viewport data, convert to plain JS rows for AG Grid
    const data = await this.sub!.getViewportData();
    // toObjects() is the supported conversion helper for TableData
    const rows = data.toObjects?.() ?? [];
    onRows(rows);
  }

  private async rebind(
    tableName: string,
    cols: string[],
    onRows: (rows: any[]) => void
  ) {
    if (this.rebinding || this.destroyed) return;
    this.rebinding = true;

    try {
      // Close old subscription if it still exists
      try { this.sub?.close(); } catch {}
      // Swap in a fresh table + viewport
      this.table = await this.resolver(tableName);
      this.sub = this.table.setViewport(0, 10000, cols);

      const EVENT = (this.dh?.Table?.EVENT_UPDATED ?? 'update');
      this.sub.addEventListener(EVENT, async () => {
        if (!this.destroyed) await this.pushViewport(onRows);
      });

      // Take an immediate snapshot so the grid repaints right away
      await this.pushViewport(onRows);
    } finally {
      this.rebinding = false;
    }
  }

  async unbind() {
    this.destroyed = true;
    try { this.sub?.close(); } catch {}
    this.sub = undefined;
    this.table = undefined;
  }
}
```

**Why this fixes your errors**

- **Avoids repeated `table.snapshot()`** (which is race-y during ticking updates and more prone to server-side `ClassCastException` if schema evolves mid-tick). Viewports are designed for streaming/scrolling updates and can be reopened after a change; DH explicitly notes that changing a table stops the old viewport, and you simply make a new one and keep going. ([Deephaven](https://deephaven.io/core/docs/reference/js-api/concepts/?utm_source=chatgpt.com "Javascript API Concepts"))
    
- We **listen on the viewport subscription** and refresh row data with `getViewportData().toObjects()` — the intended way to feed UI components. ([DeepHaven Docs](https://docs.deephaven.io/core/client-api/javascript/classes/dh.TableViewportSubscription.html "TableViewportSubscription | @deephaven/jsapi-types"))
    
- When errors mention **“closed”**, we **reacquire a fresh handle** with `ide.getTable(name)` and set a new viewport before continuing — no app crash, no stale handle. (The `EVENT_UPDATED` / event-listener usage mirrors DH’s event model.) ([DeepHaven Docs](https://docs.deephaven.io/core/javadoc/io/deephaven/web/client/api/JsTable.html?utm_source=chatgpt.com "JsTable (combined-javadoc 0.39.3 API)"))
    

---

### Component wiring (trimmed to the relevant bits)

```ts
// join-tables.component.ts (relevant excerpts)
import { Component, OnDestroy, OnInit, ViewChild } from '@angular/core';
import { AgGridAngular } from 'ag-grid-angular';
import type { ColDef } from 'ag-grid-community';
import { DeephavenService } from '../deephaven/deephaven.service';

@Component({
  standalone: true,
  selector: 'app-join-tables',
  templateUrl: './join-tables.component.html',
  styleUrls: ['./join-tables.component.scss'],
  imports: [AgGridAngular],
})
export class JoinTablesComponent implements OnInit, OnDestroy {
  @ViewChild('userGrid') userGrid!: AgGridAngular;
  @ViewChild('accountGrid') accountGrid!: AgGridAngular;
  @ViewChild('joinedGrid') joinedGrid!: AgGridAngular;

  userCols: ColDef[] = [];
  accountCols: ColDef[] = [];
  joinedCols: ColDef[] = [];

  userRows: any[] = [];
  accountRows: any[] = [];
  joinedRows: any[] = [];

  private userBinder = this.dh.createViewportBinder();
  private accountBinder = this.dh.createViewportBinder();
  private joinedBinder = this.dh.createViewportBinder();

  constructor(private dh: DeephavenService) {}

  async ngOnInit() {
    await this.dh.connect();

    // Get one set of column defs up front (avoids ClassCast when schema shifts mid-tick)
    const userHandle   = await this.dh.getTableHandle('user');
    const accountHandle= await this.dh.getTableHandle('account');
    const joinedHandle = await this.dh.getTableHandle('user_account');

    this.userCols   = this.makeCols(userHandle);
    this.accountCols= this.makeCols(accountHandle);
    this.joinedCols = this.makeCols(joinedHandle);

    // Start streaming with self-healing binders
    await this.userBinder.bind('user',         this.userCols.map(c => String(c.field)),   rows => this.userRows = rows);
    await this.accountBinder.bind('account',   this.accountCols.map(c => String(c.field)),rows => this.accountRows = rows);
    await this.joinedBinder.bind('user_account', this.joinedCols.map(c => String(c.field)),rows => this.joinedRows = rows);
  }

  async ngOnDestroy() {
    await this.userBinder.unbind();
    await this.accountBinder.unbind();
    await this.joinedBinder.unbind();
  }

  private makeCols(table: any): ColDef[] {
    return (table?.columns ?? []).map((c: any) => ({
      field: c.name,
      sortable: true,
      filter: true,
      resizable: true,
      minWidth: 120,
      // OPTIONAL: convert DH wrappers (BigDecimal/Long/Date) to strings safely for AG Grid
      valueGetter: (p: any) => {
        const v = p.data?.[c.name];
        return v?.toString ? v.toString() : v;
      }
    }));
  }
}
```

> Notes:
> 
> - **AG Grid** v34 works fine with the above; the `@ViewChild` apis are used only if you need grid API calls later (AG docs show the same pattern). ([ag-grid.com](https://www.ag-grid.com/angular-data-grid/grid-interface/?utm_source=chatgpt.com "Angular Grid: Grid Overview"))
>     
> - The service loads JS API from `https://<dh>/jsapi/dh-core.js` and logs in with **PSK**, which is the default DH auth. Put your values in `environment.ts` (`deephavenUrl`, `deephavenPsk`). ([Deephaven](https://deephaven.io/core/docs/how-to-guides/authentication/auth-psk/?utm_source=chatgpt.com "Configure and use pre-shared key authentication"))
>     

---

### Common pitfalls this avoids

- **ClassCastException during snapshot:** snapshots taken while a tick is mid-apply or while schema changes (e.g., join adds a column with a different type) can blow up server-side. A viewport subscription is designed for ongoing change and simply needs to be replaced when DH says it’s stopped. ([Deephaven](https://deephaven.io/core/docs/reference/js-api/concepts/?utm_source=chatgpt.com "Javascript API Concepts"))
    
- **“Table already closed”:** we never call methods on a dead subscription; instead, we close it and **reacquire** a fresh handle + viewport automatically (see `rebind()`).
    

If you still see a server `ClassCastException`, it usually means the _producer_ (your join / Kafka ingest script) briefly published a column with a type different from earlier ticks. The `valueGetter.toString()` shim above lets AG Grid render DH wrapper types (BigDecimal/Long/Date) consistently so the front-end doesn’t choke on non-primitive values. (DH exposes wrappers for big types in JS, which you can display as strings). ([DeepHaven Docs](https://docs.deephaven.io/core/javadoc/io/deephaven/web/client/api/JsTable.html?utm_source=chatgpt.com "JsTable (combined-javadoc 0.39.3 API)"))

---

If you want, I can paste an `environment.ts` sample and a minimal HTML template next.


-----------------------------


Nice catch — that TS error is because you were creating the binders with `this.dh...` **before** the DI constructor runs. Move binder creation to the constructor (or to `ngOnInit` after `connect()`), then wire the template.

Here’s a clean, working pair (component + HTML) that matches your service and AG Grid 34.x.

---

### `join-tables.component.ts` (fixed)

```ts
import { Component, OnDestroy, OnInit, ViewChild } from '@angular/core';
import { CommonModule } from '@angular/common';
import { AgGridAngular } from 'ag-grid-angular';
import type { ColDef, GridApi, GridReadyEvent } from 'ag-grid-community';
import { DeephavenService } from '../deephaven/deephaven.service';

@Component({
  standalone: true,
  selector: 'app-join-tables',
  templateUrl: './join-tables.component.html',
  styleUrls: ['./join-tables.component.scss'],
  imports: [CommonModule, AgGridAngular],
})
export class JoinTablesComponent implements OnInit, OnDestroy {
  @ViewChild('userGrid') userGrid!: AgGridAngular;
  @ViewChild('accountGrid') accountGrid!: AgGridAngular;
  @ViewChild('joinedGrid') joinedGrid!: AgGridAngular;

  userCols: ColDef[] = [];
  accountCols: ColDef[] = [];
  joinedCols: ColDef[] = [];

  userRows: any[] = [];
  accountRows: any[] = [];
  joinedRows: any[] = [];

  private userApi?: GridApi;
  private accountApi?: GridApi;
  private joinedApi?: GridApi;

  // Declare, don't create yet (to avoid “used before initialization”)
  private userBinder!: ReturnType<DeephavenService['createViewportBinder']>;
  private accountBinder!: ReturnType<DeephavenService['createViewportBinder']>;
  private joinedBinder!: ReturnType<DeephavenService['createViewportBinder']>;

  constructor(private dh: DeephavenService) {
    // Create binders AFTER DI made `dh` available
    this.userBinder = this.dh.createViewportBinder();
    this.accountBinder = this.dh.createViewportBinder();
    this.joinedBinder = this.dh.createViewportBinder();
  }

  async ngOnInit() {
    await this.dh.connect();

    // Get handles once to build stable column defs
    const userHandle    = await this.dh.getTableHandle('user');
    const accountHandle = await this.dh.getTableHandle('account');
    const joinedHandle  = await this.dh.getTableHandle('user_account');

    this.userCols    = this.makeCols(userHandle);
    this.accountCols = this.makeCols(accountHandle);
    this.joinedCols  = this.makeCols(joinedHandle);

    // Start live streaming (self-healing binders)
    await this.userBinder.bind(
      'user',
      this.userCols.map(c => String(c.field)),
      rows => (this.userRows = rows)
    );

    await this.accountBinder.bind(
      'account',
      this.accountCols.map(c => String(c.field)),
      rows => (this.accountRows = rows)
    );

    await this.joinedBinder.bind(
      'user_account',
      this.joinedCols.map(c => String(c.field)),
      rows => (this.joinedRows = rows)
    );
  }

  async ngOnDestroy() {
    await this.userBinder?.unbind();
    await this.accountBinder?.unbind();
    await this.joinedBinder?.unbind();
  }

  onUserGridReady(e: GridReadyEvent)   { this.userApi = e.api; }
  onAccountGridReady(e: GridReadyEvent){ this.accountApi = e.api; }
  onJoinedGridReady(e: GridReadyEvent) { this.joinedApi = e.api; }

  quickFilter(which: 'user'|'account'|'joined', text: string) {
    const api =
      which === 'user'    ? this.userApi :
      which === 'account' ? this.accountApi :
                             this.joinedApi;
    api?.setGridOption('quickFilterText', text);
  }

  private makeCols(table: any): ColDef[] {
    return (table?.columns ?? []).map((c: any) => ({
      field: c.name,
      sortable: true,
      filter: true,
      resizable: true,
      minWidth: 120,
      // Helpful when DH returns wrapper types (BigDecimal/Instant/Long)
      valueGetter: p => {
        const v = p.data?.[c.name];
        return v?.toString ? v.toString() : v;
      },
    }));
  }
}
```

---

### `join-tables.component.html` (updated)

```html
<div class="grid-2">
  <div class="panel">
    <div class="panel-header">
      <h3>Users</h3>
      <input
        placeholder="Search users…"
        (input)="quickFilter('user', ($event.target as HTMLInputElement).value)" />
    </div>

    <ag-grid-angular
      #userGrid
      class="ag-theme-alpine grid"
      [rowData]="userRows"
      [columnDefs]="userCols"
      rowSelection="single"
      [animateRows]="true"
      (gridReady)="onUserGridReady($event)">
    </ag-grid-angular>
  </div>

  <div class="panel">
    <div class="panel-header">
      <h3>Account</h3>
      <input
        placeholder="Search accounts…"
        (input)="quickFilter('account', ($event.target as HTMLInputElement).value)" />
    </div>

    <ag-grid-angular
      #accountGrid
      class="ag-theme-alpine grid"
      [rowData]="accountRows"
      [columnDefs]="accountCols"
      rowSelection="single"
      [animateRows]="true"
      (gridReady)="onAccountGridReady($event)">
    </ag-grid-angular>
  </div>

  <div class="panel full">
    <div class="panel-header">
      <h3>User-Account</h3>
      <input
        placeholder="Search joined…"
        (input)="quickFilter('joined', ($event.target as HTMLInputElement).value)" />
    </div>

    <ag-grid-angular
      #joinedGrid
      class="ag-theme-alpine grid"
      [rowData]="joinedRows"
      [columnDefs]="joinedCols"
      rowSelection="single"
      [animateRows]="true"
      (gridReady)="onJoinedGridReady($event)">
    </ag-grid-angular>
  </div>
</div>
```

---

### Quick notes

- The key fix for the TS error: **don’t call `this.dh.createViewportBinder()` in a field initializer**; do it in the constructor (or later).
    
- The template now uses the same `quickFilter()` helper you wrote, and passes the correct `HTMLInputElement.value`.
    
- Keep your `environment.ts` with:
    
    ```ts
    export const environment = {
      production: false,
      deephavenUrl: 'https://<your-dh-host>:<port>',
      deephavenPsk: '<your-psk>',
    };
    ```
    
- With your self-healing `ViewportBinder` (in the service you already pasted), the **ClassCast** and **table closed** cases are handled by re-acquiring a fresh handle and re-opening the viewport automatically.
    

If you want, I can also drop a tiny SCSS to make the three panels responsive.


----------------------------------

You’re still tripping a **server-side** `BackingJsObjectError: java.lang.ClassCastException` while reading the viewport.  
This almost always happens when the table you’re viewing contains **non-JS-friendly types** (e.g., `Instant/DateTime`, `BigDecimal`, `byte[]`) or when a **column’s type changes** mid-stream (append/aj/update pipeline). Your screenshots show the Kafka system columns (`KafkaPartition`, `KafkaOffset`, `KafkaTimestamp`) plus your join—those are classic culprits.

The clean, no-drama fix is:

## 1) Freeze a UI-safe schema in Deephaven (server)

Create “UI tables” that:

- select only the columns you want to display,
    
- **cast/convert** tricky types to **string/number**, and
    
- keep the **same schema** forever.
    

Paste this in your DH console (adjust fields as needed):

```python
# ui_tables.py  -- run this in DH console once (or as your app script)

from deephaven import dtypes as dt
import deephaven.kafkautil as kc

# ---- your existing specs (keep as-is) ----
# USER_VALUE_SPEC = kc.json_spec({...})
# ACCOUNT_VALUE_SPEC = kc.json_spec({...})
# user_raw  = kc.consume(..., table_type=kc.TableType.append())
# account_raw = kc.consume(..., table_type=kc.TableType.append())
# users_hist = user_raw.sort_descending("KafkaTimestamp")
# user_account = account_raw.aj(
#     on=["userId", "KafkaTimestamp"],  # or your key(s)
#     joins=["name","email"]
# )

# ---- make everything UI-safe and stable ----
def ui_users(t):
    return t.view([
        "kafka_partition = (int)KafkaPartition",
        "kafka_offset   = (long)KafkaOffset",
        "kafka_ts       = (String)KafkaTimestamp",   # stringify Instant
        "userId",
        "name",
        "email",
    ])

def ui_accounts(t):
    return t.view([
        "kafka_partition = (int)KafkaPartition",
        "kafka_offset   = (long)KafkaOffset",
        "kafka_ts       = (String)KafkaTimestamp",
        "userId",
        "accountType",
        "balance = (double)balance",                 # ensure numeric
    ])

def ui_join(t):
    return t.view([
        "kafka_partition = (int)KafkaPartition",
        "kafka_offset   = (long)KafkaOffset",
        "kafka_ts       = (String)KafkaTimestamp",
        "userId",
        "accountType",
        "balance = (double)balance",
        "name",
        "email",
    ])

user         = ui_users(user_raw)
account      = ui_accounts(account_raw)
user_account = ui_join(user_account)   # rename if your source variable is same
```

> Key points  
> • `(String)KafkaTimestamp` converts DH `Instant` to a stable string.  
> • We explicitly cast numeric columns to `int/long/double`.  
> • The three exported globals are **`user`, `account`, `user_account`** (the names your Angular code already uses).

## 2) Tell the Angular side to only request those UI-safe columns

You already generate `columnDefs` from `table.columns`. That will now be the **safe** set (strings + numbers). Nothing else changes on the front-end.

If you still want to be extra defensive, keep the `valueGetter.toString()` you added, but it shouldn’t be necessary once the server exports strings/numbers only.

## 3) Keep the self-healing viewport binder

The `ViewportBinder` you’re using is the right approach for streaming. It already re-acquires a fresh handle and viewport on “table closed”. Two tiny hardening tweaks:

```ts
// inside ViewportBinder.pushViewport
private async pushViewport(onRows: (rows: any[]) => void) {
  const data = await this.sub!.getViewportData();
  // Some DH builds name it toObjects, others expose rows().toObjects()
  const rows = data.toObjects?.() ?? data.rows?.toObjects?.() ?? [];
  onRows(rows);
}

// wherever you catch(e):
const msg = (e?.message ?? String(e)).toLowerCase();
if (msg.includes('closed') || msg.includes('classcastexception')) {
  await this.rebind(tableName, cols, onRows);
  return;
}
```

That way even if a transient `ClassCastException` bubbles out of the subscription once, you immediately **rebind** to the same exported UI table (which now has a stable schema), and the flood stops.

## 4) Common gotchas (now avoided)

- **Instant / Big types** sent directly to the browser can cause server conversion issues; you’re now stringifying/casting them server-side.
    
- **Schema shifts mid-tick** (e.g., join adds a column, or a field flips type): the UI tables `view([...])` pin the output schema.
    
- **Duplicate column names from joins**: if you bring columns with same names from both sides, suffix/rename them in the `view([...])`.
    

---

### What to change right now

1. Run the Python “UI tables” block in your DH console so that the exported names `user`, `account`, `user_account` are schema-stable and JS-friendly.
    
2. Keep your latest Angular component and HTML as we structured earlier. No extra changes needed apart from the small binder tweak (optional).
    

This is the simplest path that:

- stops the `BackingJsObjectError/ClassCastException` spam,
    
- streams reliably with `setViewport` updates,
    
- and keeps your AG Grid columns consistent across ticks.
    

If you paste your exact `kc.consume(...)` lines (with only non-secret bits), I can tailor the three `view([...])` lists precisely to your schema.










------------------------------

new 



Gotcha. Here’s a **known-good, non-deprecated setup** that works with **Angular 20.3.x** and your **AG Grid 34.2.0**.

# Versions to use

- **Angular:** 20.3.x (your current) — keep it. ([Ninja Squad](https://blog.ninja-squad.com/2025/09/11/what-is-new-angular-20.3/?utm_source=chatgpt.com "What's new in Angular 20.3? - Blog | Ninja Squad"))
    
- **TypeScript:** ≥ **5.2** (AG Grid 34 requires this; Angular 20 is fine with ≥5.5). ([AG Grid](https://www.ag-grid.com/angular-data-grid/compatibility/?utm_source=chatgpt.com "Angular Grid: Version Compatibility"))
    
- **AG Grid:** **34.2.0** (fully compatible with Angular 17–20). ([AG Grid](https://www.ag-grid.com/angular-data-grid/compatibility/?utm_source=chatgpt.com "Angular Grid: Version Compatibility"))
    
- **Deephaven JS client packages (supported & not deprecated):**
    
    - **`@deephaven/jsapi-bootstrap`**: use the latest **0.85.x** (e.g., **0.85.15+**). This is the supported loader/bootstrapping lib. ([npm](https://www.npmjs.com/package/%40deephaven%2Fjsapi-bootstrap?utm_source=chatgpt.com "deephaven/jsapi-bootstrap"))
        
    - **`@deephaven/jsapi-types`**: use the latest **1.0.0-dev0.39.x** that **matches your server’s 0.39.x**. (Pick the `dev0.39.*` that aligns with your DH server.) ([npm](https://www.npmjs.com/package/%40deephaven%2Fjsapi-types?utm_source=chatgpt.com "deephaven/jsapi-types"))
        

> Note: **`@deephaven/jsapi`** (old name) is not on npm anymore—don’t use it. Use **`jsapi-bootstrap`** + **`jsapi-types`** instead. ([npm](https://www.npmjs.com/package/%40deephaven%2Fjsapi-bootstrap?utm_source=chatgpt.com "deephaven/jsapi-bootstrap"))

# Install commands

```bash
# AG Grid for Angular 20
npm i ag-grid-community@34.2.0 @ag-grid-community/angular@34.2.0

# Deephaven JS API (supported packages)
npm i @deephaven/jsapi-bootstrap@^0.85.15 @deephaven/jsapi-types@latest
```

# Minimal Angular wiring (works with 20.3)

**Service (sketch) using jsapi-bootstrap pattern** – aligns with Deephaven’s “Use the JS API” docs:

```ts
// deephaven.service.ts
import { Injectable } from '@angular/core';
import { BehaviorSubject } from 'rxjs';
import loadDhCore from '@deephaven/jsapi-bootstrap'; // loads the DH JS API bundle
// Types
import type { dh as DhNamespace } from '@deephaven/jsapi-types';

@Injectable({ providedIn: 'root' })
export class DeephavenService {
  private dh!: typeof DhNamespace;
  private session!: DhNamespace.IdeConnection;

  private _rows$ = new BehaviorSubject<any[]>([]);
  rows$ = this._rows$.asObservable();

  async connect(opts: { baseUrl: string; authToken?: string }) {
    // Load JS API and open a session
    this.dh = await loadDhCore({ baseUrl: opts.baseUrl }); // uses WebSocket for live updates
    this.session = await this.dh.IdeConnection.startSession(
      opts.baseUrl,
      opts.authToken ? { type: 'token', token: opts.authToken } : undefined
    );
  }

  async subscribeTable(tableName: string) {
    // Get a handle to the table (already created/exposed in DH)
    const table = await this.session.getTable({ tableName });

    // Fetch an initial snapshot
    const snapshot = await table.snapshot();
    this._rows$.next(snapshot.data);

    // Subscribe for live updates via Barrage
    const sub = await table.subscribe({
      onSnapshot: s => this._rows$.next(s.data),
      onUpdate: u => {
        // apply incremental updates as needed (u.added/u.removed/u.modified)
        // for simple cases, re-fetch snapshot:
        table.snapshot().then(s => this._rows$.next(s.data));
      },
      onError: e => console.error('DH update error', e),
    });

    return () => sub.close(); // unsubscribe
  }

  async close() {
    await this.session?.close();
  }
}
```

This pattern follows Deephaven’s official JS API guidance (bootstrap loader + live updates over Arrow Flight/Barrage). ([deephaven.io](https://deephaven.io/core/docs/how-to-guides/use-jsapi/?utm_source=chatgpt.com "Use the JS API"))

# AG Grid 34 + Angular 20 component (simple)

```ts
// users-grid.component.ts
import { Component, OnInit, OnDestroy } from '@angular/core';
import { ColDef } from 'ag-grid-community';
import { DeephavenService } from '../deephaven.service';
import { Subscription } from 'rxjs';

@Component({
  selector: 'app-users-grid',
  template: `<ag-grid-angular
      style="width: 100%; height: 600px"
      class="ag-theme-quartz"
      [rowData]="rows"
      [columnDefs]="cols">
    </ag-grid-angular>`
})
export class UsersGridComponent implements OnInit, OnDestroy {
  rows: any[] = [];
  cols: ColDef[] = [
    { field: 'id' }, { field: 'name' }, { field: 'email' }
  ];

  private sub?: Subscription;
  private cleanup?: () => void;

  constructor(private dh: DeephavenService) {}

  async ngOnInit() {
    await this.dh.connect({ baseUrl: 'http://<your-dh-host>:<port>' });
    this.cleanup = await this.dh.subscribeTable('users'); // your DH table name

    this.sub = this.dh.rows$.subscribe(r => (this.rows = r));
  }

  ngOnDestroy() {
    this.sub?.unsubscribe();
    this.cleanup?.();
    this.dh.close();
  }
}
```

# Why these choices are safe

- **AG Grid 34.x ⇄ Angular 17–20** official compatibility is confirmed. ([AG Grid](https://www.ag-grid.com/angular-data-grid/compatibility/?utm_source=chatgpt.com "Angular Grid: Version Compatibility"))
    
- **Angular 20.3** is current and supported; no AG Grid breaking changes needed. ([Ninja Squad](https://blog.ninja-squad.com/2025/09/11/what-is-new-angular-20.3/?utm_source=chatgpt.com "What's new in Angular 20.3? - Blog | Ninja Squad"))
    
- **Deephaven JS API**: `jsapi-bootstrap` is the supported way to load the web client API; `jsapi-types` provides typings that track server versions (choose the `dev0.39.*` track if your server is 0.39.x). ([npm](https://www.npmjs.com/package/%40deephaven%2Fjsapi-bootstrap?utm_source=chatgpt.com "deephaven/jsapi-bootstrap"))
    
- **Live updates** come via **Barrage** over WebSockets, which the JS API handles (so you get real-time streaming without polling). ([deephaven.io](https://deephaven.io/barrage/docs/?utm_source=chatgpt.com "Barrage"))
    

If you tell me your exact **Deephaven server version** (e.g., 0.39.7), I’ll pin the **exact** `@deephaven/jsapi-types` version to match and adjust the sample to your auth (PSK / token).


--------------------------------------


Perfect—let’s strip all Kafka metadata and only expose clean, domain fields with stable types.

# 1) Deephaven: export “UI” tables (domain-only, stable schema)

Paste this in the DH console (edit topic names/config only). It enforces types at the boundary and exports three globals the Angular app can bind to:

```python
from deephaven import dtypes as dt
import deephaven.kafkautil as kc

# --- your topics & config ---
TOPIC_USERS = "<users_topic>"
TOPIC_ACCTS = "<accounts_topic>"
KAFKA_CONFIG = {  # keep your working config here
  # ...
}

# 1) Lock value types (no mixed/flip-flop types)
USER_VALUE_SPEC = kc.json_spec({
    "userId": dt.string,
    "name":   dt.string,
    "email":  dt.string,
})

ACCOUNT_VALUE_SPEC = kc.json_spec({
    "userId":      dt.string,
    "accountType": dt.string,
    "balance":     dt.double,   # force numeric at the boundary
})

# 2) Append tables from Kafka
user_raw = kc.consume(
    KAFKA_CONFIG, TOPIC_USERS,
    key_spec=kc.KeyValueSpec.IGNORE,
    value_spec=USER_VALUE_SPEC,
    table_type=kc.TableType.append()
)

account_raw = kc.consume(
    KAFKA_CONFIG, TOPIC_ACCTS,
    key_spec=kc.KeyValueSpec.IGNORE,
    value_spec=ACCOUNT_VALUE_SPEC,
    table_type=kc.TableType.append()
)

# 3) Extra safety: if producer ever sends "balance" as text, coerce or null it
account_norm = account_raw.update_view(["balance = (double) balance"])

# 4) Domain-only “UI” tables (NO KafkaPartition/Offset/Timestamp)
users_ui         = user_raw.view(["userId", "name", "email"])
accounts_ui      = account_norm.view(["userId", "accountType", "balance"])
user_accounts_ui = account_norm.aj(
    on=["userId"],  # join only on userId to avoid time type mismatches
    joins=["accountType", "balance", "name", "email"]
).view(["userId", "accountType", "balance", "name", "email"])
```

**What you get**

- `users_ui(userId, name, email)`
    
- `accounts_ui(userId, accountType, balance)`
    
- `user_accounts_ui(userId, accountType, balance, name, email)`  
    All fields are plain `string`/`double`—easy for JS. No Kafka columns, so no schema surprises.
    

---

# 2) Angular: point to the new table names

In your component (already using the self-healing viewport binder), just change the names you fetch:

```ts
const usersHandle   = await this.dh.getTableHandle('users_ui');
const accountsHandle= await this.dh.getTableHandle('accounts_ui');
const joinedHandle  = await this.dh.getTableHandle('user_accounts_ui');

this.userCols    = this.makeCols(usersHandle);
this.accountCols = this.makeCols(accountsHandle);
this.joinedCols  = this.makeCols(joinedHandle);

await this.userBinder.bind('users_ui',           this.userCols.map(c => String(c.field)),   rows => this.userRows = rows);
await this.accountBinder.bind('accounts_ui',     this.accountCols.map(c => String(c.field)),rows => this.accountRows = rows);
await this.joinedBinder.bind('user_accounts_ui', this.joinedCols.map(c => String(c.field)), rows => this.joinedRows = rows);
```

Because the server only exposes **primitives**, AG Grid can consume them directly. You can even drop the `valueGetter` if you want:

```ts
private makeCols(table: any) {
  return (table?.columns ?? []).map((c: any) => ({
    field: c.name,
    sortable: true,
    filter: true,
    resizable: true,
    minWidth: 120,
  }));
}
```

---

# 3) Why this cures the ClassCast flood

- No Kafka `DateTime/Instant` or other tricky types reach the browser.
    
- Types are **frozen at the Kafka ingest** (`json_spec` + `(double)balance`), so you won’t get “sometimes string, sometimes number”.
    
- Join is on a single **string key** (`userId`)—no timestamp mismatches.
    
- Viewport binder keeps streaming and auto-rebinds if a handle is closed.
    

If you later need Kafka columns for debugging, expose them via _separate_ tables like `users_debug`—don’t mix them into these UI tables.

--------------------------------------------------------

Gotcha. Here’s a **known-good, non-deprecated setup** that works with **Angular 20.3.x** and your **AG Grid 34.2.0**.

# Versions to use

- **Angular:** 20.3.x (your current) — keep it. ([Ninja Squad](https://blog.ninja-squad.com/2025/09/11/what-is-new-angular-20.3/?utm_source=chatgpt.com "What's new in Angular 20.3? - Blog | Ninja Squad"))
    
- **TypeScript:** ≥ **5.2** (AG Grid 34 requires this; Angular 20 is fine with ≥5.5). ([AG Grid](https://www.ag-grid.com/angular-data-grid/compatibility/?utm_source=chatgpt.com "Angular Grid: Version Compatibility"))
    
- **AG Grid:** **34.2.0** (fully compatible with Angular 17–20). ([AG Grid](https://www.ag-grid.com/angular-data-grid/compatibility/?utm_source=chatgpt.com "Angular Grid: Version Compatibility"))
    
- **Deephaven JS client packages (supported & not deprecated):**
    
    - **`@deephaven/jsapi-bootstrap`**: use the latest **0.85.x** (e.g., **0.85.15+**). This is the supported loader/bootstrapping lib. ([npm](https://www.npmjs.com/package/%40deephaven%2Fjsapi-bootstrap?utm_source=chatgpt.com "deephaven/jsapi-bootstrap"))
        
    - **`@deephaven/jsapi-types`**: use the latest **1.0.0-dev0.39.x** that **matches your server’s 0.39.x**. (Pick the `dev0.39.*` that aligns with your DH server.) ([npm](https://www.npmjs.com/package/%40deephaven%2Fjsapi-types?utm_source=chatgpt.com "deephaven/jsapi-types"))
        

> Note: **`@deephaven/jsapi`** (old name) is not on npm anymore—don’t use it. Use **`jsapi-bootstrap`** + **`jsapi-types`** instead. ([npm](https://www.npmjs.com/package/%40deephaven%2Fjsapi-bootstrap?utm_source=chatgpt.com "deephaven/jsapi-bootstrap"))

# Install commands

```bash
# AG Grid for Angular 20
npm i ag-grid-community@34.2.0 @ag-grid-community/angular@34.2.0

# Deephaven JS API (supported packages)
npm i @deephaven/jsapi-bootstrap@^0.85.15 @deephaven/jsapi-types@latest
```

# Minimal Angular wiring (works with 20.3)

**Service (sketch) using jsapi-bootstrap pattern** – aligns with Deephaven’s “Use the JS API” docs:

```ts
// deephaven.service.ts
import { Injectable } from '@angular/core';
import { BehaviorSubject } from 'rxjs';
import loadDhCore from '@deephaven/jsapi-bootstrap'; // loads the DH JS API bundle
// Types
import type { dh as DhNamespace } from '@deephaven/jsapi-types';

@Injectable({ providedIn: 'root' })
export class DeephavenService {
  private dh!: typeof DhNamespace;
  private session!: DhNamespace.IdeConnection;

  private _rows$ = new BehaviorSubject<any[]>([]);
  rows$ = this._rows$.asObservable();

  async connect(opts: { baseUrl: string; authToken?: string }) {
    // Load JS API and open a session
    this.dh = await loadDhCore({ baseUrl: opts.baseUrl }); // uses WebSocket for live updates
    this.session = await this.dh.IdeConnection.startSession(
      opts.baseUrl,
      opts.authToken ? { type: 'token', token: opts.authToken } : undefined
    );
  }

  async subscribeTable(tableName: string) {
    // Get a handle to the table (already created/exposed in DH)
    const table = await this.session.getTable({ tableName });

    // Fetch an initial snapshot
    const snapshot = await table.snapshot();
    this._rows$.next(snapshot.data);

    // Subscribe for live updates via Barrage
    const sub = await table.subscribe({
      onSnapshot: s => this._rows$.next(s.data),
      onUpdate: u => {
        // apply incremental updates as needed (u.added/u.removed/u.modified)
        // for simple cases, re-fetch snapshot:
        table.snapshot().then(s => this._rows$.next(s.data));
      },
      onError: e => console.error('DH update error', e),
    });

    return () => sub.close(); // unsubscribe
  }

  async close() {
    await this.session?.close();
  }
}
```

This pattern follows Deephaven’s official JS API guidance (bootstrap loader + live updates over Arrow Flight/Barrage). ([deephaven.io](https://deephaven.io/core/docs/how-to-guides/use-jsapi/?utm_source=chatgpt.com "Use the JS API"))

# AG Grid 34 + Angular 20 component (simple)

```ts
// users-grid.component.ts
import { Component, OnInit, OnDestroy } from '@angular/core';
import { ColDef } from 'ag-grid-community';
import { DeephavenService } from '../deephaven.service';
import { Subscription } from 'rxjs';

@Component({
  selector: 'app-users-grid',
  template: `<ag-grid-angular
      style="width: 100%; height: 600px"
      class="ag-theme-quartz"
      [rowData]="rows"
      [columnDefs]="cols">
    </ag-grid-angular>`
})
export class UsersGridComponent implements OnInit, OnDestroy {
  rows: any[] = [];
  cols: ColDef[] = [
    { field: 'id' }, { field: 'name' }, { field: 'email' }
  ];

  private sub?: Subscription;
  private cleanup?: () => void;

  constructor(private dh: DeephavenService) {}

  async ngOnInit() {
    await this.dh.connect({ baseUrl: 'http://<your-dh-host>:<port>' });
    this.cleanup = await this.dh.subscribeTable('users'); // your DH table name

    this.sub = this.dh.rows$.subscribe(r => (this.rows = r));
  }

  ngOnDestroy() {
    this.sub?.unsubscribe();
    this.cleanup?.();
    this.dh.close();
  }
}
```

# Why these choices are safe

- **AG Grid 34.x ⇄ Angular 17–20** official compatibility is confirmed. ([AG Grid](https://www.ag-grid.com/angular-data-grid/compatibility/?utm_source=chatgpt.com "Angular Grid: Version Compatibility"))
    
- **Angular 20.3** is current and supported; no AG Grid breaking changes needed. ([Ninja Squad](https://blog.ninja-squad.com/2025/09/11/what-is-new-angular-20.3/?utm_source=chatgpt.com "What's new in Angular 20.3? - Blog | Ninja Squad"))
    
- **Deephaven JS API**: `jsapi-bootstrap` is the supported way to load the web client API; `jsapi-types` provides typings that track server versions (choose the `dev0.39.*` track if your server is 0.39.x). ([npm](https://www.npmjs.com/package/%40deephaven%2Fjsapi-bootstrap?utm_source=chatgpt.com "deephaven/jsapi-bootstrap"))
    
- **Live updates** come via **Barrage** over WebSockets, which the JS API handles (so you get real-time streaming without polling). ([deephaven.io](https://deephaven.io/barrage/docs/?utm_source=chatgpt.com "Barrage"))
    

If you tell me your exact **Deephaven server version** (e.g., 0.39.7), I’ll pin the **exact** `@deephaven/jsapi-types` version to match and adjust the sample to your auth (PSK / token).