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