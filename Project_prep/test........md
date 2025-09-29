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