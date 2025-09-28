You’re seeing that storm because the frontend is opening **multiple viewports on the same DH global** (every bind/reattach from HMR or re-renders adds another), so when you re-run the Python cell and DH swaps the globals, all those stale viewports start firing `snapshot()` → `ClassCastException` → `Table already closed`.

Fix = make binding **single-flight per table variable** and **broadcast** updates to any listeners. That way there is only ever **one** viewport per DH global. Also make the binder self-healing (reattach once when the table is swapped) and make unsubscribe deterministic.

Below is an end-to-end update that addresses all the issues you’ve hit:

- PSK login (unchanged).
    
- A **TableHub** inside the service that:
    
    - de-duplicates by `varName`
        
    - keeps exactly **one** viewport open per table
        
    - snapshots once and **broadcasts** rows to all subscribers
        
    - self-heals on “closed” errors without flooding
        
    - reference-counts and closes the viewport when the last subscriber unsubscribes
        
- A minimal `LiveTableComponent` using the hub; it never calls `snapshot()` directly, so there’s no attach storm.
    
- Server side: keep your **top-level** DH cell (no helpers) that recreates `user`, `account`, `user_account` each run. The hub will reattach when that happens.
    

---

## 1) Deephaven service (PSK + TableHub)

`src/app/deephaven/deephaven.service.ts`

```ts
import { Injectable } from '@angular/core';
import { environment } from '../../environments/environment';

type DhNS = any;
type DhClient = any;
type DhIDE = any;
export type DhTable = any;

type RowsListener = (rows: any[]) => void;

interface HubEntry {
  varName: string;
  table?: DhTable;
  viewport?: any;
  cols: string[];
  listeners: Set<RowsListener>;
  refCount: number;
  generation: number;
  reconnecting: boolean;
  destroyed: boolean;
}

@Injectable({ providedIn: 'root' })
export class DeephavenService {
  private dh!: DhNS;
  private client!: DhClient;
  private ide!: DhIDE;
  private ready = false;

  getDhNs() { return this.dh; }
  get isReady() { return this.ready; }

  /** PSK login – same as before */
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

  async getTableHandle(varName: string): Promise<DhTable> {
    if (!this.ready) await this.connect();
    if (!/^[A-Za-z_][A-Za-z0-9_]*$/.test(varName)) {
      throw new Error(`Invalid table variable name: "${varName}"`);
    }
    return this.ide.getTable(varName);
  }

  // --------------------------- TableHub ---------------------------

  private hub = new Map<string, HubEntry>();

  /**
   * Subscribe to a DH globals() table by variable name.
   * Ensures ONLY ONE viewport per table variable.
   * Returns an unsubscribe() that decrements the ref and cleans up if last.
   */
  async subscribe(
    varName: string,
    onRows: RowsListener,
    opts?: { offset?: number; size?: number }
  ): Promise<() => Promise<void>> {
    await this.connect();
    let entry = this.hub.get(varName);
    if (!entry) {
      entry = {
        varName,
        cols: [],
        listeners: new Set<RowsListener>(),
        refCount: 0,
        generation: 0,
        reconnecting: false,
        destroyed: false,
      };
      this.hub.set(varName, entry);
      // Lazy attach on first subscriber
    }

    entry.refCount++;
    entry.listeners.add(onRows);

    if (!entry.table) {
      await this.attach(entry, opts);
    } else if (opts) {
      await this.setWindow(entry, opts.offset ?? 0, opts.size ?? 300, onRows);
    } else {
      // push a snapshot to the new listener immediately
      await this.pushSnapshot(entry, onRows).catch(() => {});
    }

    // Unsubscribe
    return async () => {
      if (!this.hub.has(varName)) return;
      const e = this.hub.get(varName)!;
      e.listeners.delete(onRows);
      e.refCount = Math.max(0, e.refCount - 1);
      if (e.refCount === 0) {
        await this.detach(e);
        this.hub.delete(varName);
      }
    };
  }

  // ---------- internals ----------

  private isClosed(e: any): boolean {
    const m = (e?.message ?? String(e)).toLowerCase();
    return m.includes('closed') || m.includes('table already closed');
  }

  private toRows(entry: HubEntry, objects: any[]): any[] {
    return objects.map(o => (Array.isArray(o) ? o : entry.cols.map(c => o?.[c])));
  }

  private eventNames() {
    const ns = this.getDhNs();
    return {
      EVU: ns?.Table?.EVENT_UPDATED ?? 'update',
      EVS: ns?.Table?.EVENT_SCHEMA_CHANGED ?? 'schema_changed',
    };
  }

  private async attach(entry: HubEntry, opts?: { offset?: number; size?: number }) {
    const { EVU, EVS } = this.eventNames();
    const myGen = ++entry.generation;
    const offset = opts?.offset ?? 0;
    const size = opts?.size ?? 300;

    // (Re)acquire table
    entry.table = await this.getTableHandle(entry.varName);
    entry.cols = (entry.table.columns ?? []).map((c: any) => c.name);

    // Open viewport
    entry.viewport = await entry.table.setViewport(offset, offset + size - 1, entry.cols);

    // First snapshot & broadcast
    await this.broadcastSnapshot(entry).catch(async (e) => {
      if (this.isClosed(e)) await this.reattach(entry);
    });

    // Bind listeners (single instance per varName)
    const onUpdate = async () => {
      if (entry.destroyed || myGen !== entry.generation) return;
      try {
        await this.broadcastSnapshot(entry);
      } catch (e) {
        if (this.isClosed(e)) await this.reattach(entry);
      }
    };
    const onSchema = async () => {
      if (entry.destroyed || myGen !== entry.generation) return;
      try {
        entry.cols = (entry.table!.columns ?? []).map((c: any) => c.name);
        await entry.table!.setViewport(offset, offset + size - 1, entry.cols);
        await this.broadcastSnapshot(entry);
      } catch (e) {
        if (this.isClosed(e)) await this.reattach(entry);
      }
    };

    // Store so we can remove later
    (entry as any)._onUpdate = onUpdate;
    (entry as any)._onSchema = onSchema;

    entry.table.addEventListener(EVU, onUpdate);
    entry.table.addEventListener(EVS, onSchema);
  }

  private async detach(entry: HubEntry) {
    entry.destroyed = true;
    try {
      const { EVU, EVS } = this.eventNames();
      if (entry.table && (entry as any)._onUpdate) {
        entry.table.removeEventListener?.(EVU, (entry as any)._onUpdate);
      }
      if (entry.table && (entry as any)._onSchema) {
        entry.table.removeEventListener?.(EVS, (entry as any)._onSchema);
      }
    } catch {}
    try { await entry.viewport?.close?.(); } catch {}
    entry.viewport = undefined;
    // Do NOT call table.close(); it's a server global.
    entry.table = undefined;
  }

  private async reattach(entry: HubEntry) {
    if (entry.reconnecting || entry.destroyed) return;
    entry.reconnecting = true;
    try {
      await this.detach(entry);
      await new Promise(r => setTimeout(r, 150)); // let DH finish swapping globals
      await this.attach(entry);
    } finally {
      entry.reconnecting = false;
    }
  }

  private async pushSnapshot(entry: HubEntry, toOne?: RowsListener) {
    if (!entry.table) return;
    const snap = await entry.table.snapshot(entry.cols);
    const objs = snap.toObjects?.() ?? snap;
    const rows = this.toRows(entry, objs);
    if (toOne) toOne(rows);
  }

  private async broadcastSnapshot(entry: HubEntry) {
    if (!entry.table || entry.listeners.size === 0) return;
    const snap = await entry.table.snapshot(entry.cols);
    const objs = snap.toObjects?.() ?? snap;
    const rows = this.toRows(entry, objs);
    entry.listeners.forEach(l => l(rows));
  }

  private async setWindow(entry: HubEntry, offset: number, size: number, toOne?: RowsListener) {
    if (!entry.table) return;
    await entry.table.setViewport(offset, offset + size - 1, entry.cols);
    if (toOne) await this.pushSnapshot(entry, toOne);
  }
}
```

**Why this stops the error flood**

- Each table variable (`'user'`, `'account'`, `'user_account'`) has **one** viewport and **one** set of listeners in the hub, regardless of how many components subscribe.
    
- On DH cell re-run, stale handles throw “closed” once → hub **reattaches once** and resumes.
    
- Late events from old generations are ignored.
    

---

## 2) Live component (simple consumer of the hub)

`src/app/live-table/live-table.component.ts`

```ts
import { Component, Input, OnInit, OnDestroy, ChangeDetectionStrategy, ChangeDetectorRef } from '@angular/core';
import { CommonModule } from '@angular/common';
import { DeephavenService } from '../deephaven/deephaven.service';

@Component({
  selector: 'app-live-table',
  standalone: true,
  imports: [CommonModule],
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <section class="card">
      <header class="bar">
        <h3><span class="mono">{{ varName }}</span></h3>
        <div *ngIf="error" class="error">{{ error }}</div>
      </header>

      <div class="grid" *ngIf="!error">
        <div class="thead" *ngIf="cols.length">
          <div class="th" *ngFor="let c of cols; trackBy: trackCol">{{ c }}</div>
        </div>
        <div class="tbody">
          <div class="tr" *ngFor="let r of rows; trackBy: trackRow">
            <div class="td" *ngFor="let v of r; trackBy: trackCell">{{ v }}</div>
          </div>
        </div>
      </div>
    </section>
  `,
  styles: [`
    :host{display:block}.card{border:1px solid #2a2a2a;border-radius:.75rem;background:#0f0f0f;color:#fff}
    .bar{display:flex;justify-content:space-between;align-items:center;padding:.75rem 1rem;border-bottom:1px solid #242424}
    .mono{font-family:ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, "Liberation Mono", monospace}
    .grid{display:grid;grid-template-rows:auto 1fr;gap:.5rem}
    .thead,.tr{display:grid;grid-auto-flow:column;grid-auto-columns:minmax(140px,1fr)}
    .th{position:sticky;top:0;background:#111;font-weight:600;padding:.5rem;z-index:1}
    .tbody{max-height:40dvh;overflow:auto}
    .td{padding:.25rem .5rem;border-top:1px solid #242424;white-space:nowrap;overflow:hidden;text-overflow:ellipsis}
    .error{color:#ff6b6b}
  `]
})
export class LiveTableComponent implements OnInit, OnDestroy {
  /** DH globals() variable name */
  @Input({ required: true }) varName!: string;
  @Input() pageSize = 300;

  cols: string[] = [];
  rows: any[] = [];
  error: string | null = null;

  private unsub?: () => Promise<void>;

  constructor(private dh: DeephavenService, private cdr: ChangeDetectorRef) {}

  async ngOnInit() {
    try {
      await this.dh.connect();
      // Pull columns once (optional)
      const t = await this.dh.getTableHandle(this.varName);
      this.cols = (t.columns ?? []).map((c: any) => c.name);

      // Subscribe via the hub (single viewport per table)
      this.unsub = await this.dh.subscribe(this.varName, rows => {
        this.rows = rows; this.cdr.markForCheck();
      }, { offset: 0, size: this.pageSize });
    } catch (e: any) {
      this.error = e?.message ?? String(e);
      this.cdr.markForCheck();
    }
  }

  async ngOnDestroy() { try { await this.unsub?.(); } catch {} }

  trackCol = (_: number, c: string) => c;
  trackRow = (_: number, r: any[]) => r?.[0] ?? _;
  trackCell = (_: number, _v: any) => _;
}
```

Use it three times in your page:

```html
<app-live-table varName="user"></app-live-table>
<app-live-table varName="account"></app-live-table>
<app-live-table varName="user_account"></app-live-table>
```

No ag-grid needed for this test; once stable, you can feed `rows`/`cols` into ag-grid as before – the hub makes the data flow identical (just remove any direct `snapshot()` calls from components).

---

## 3) Deephaven Python cell (top-level, unconditional)

Keep using the **single top-level** cell you now have (no helper functions, no `if in globals()` guards), e.g.:

```python
from deephaven.stream.kafka import consumer as kc
from deephaven import dtypes as dt

# Clean old names
for _n in ("user_account","users_hist","user","account","user_raw","account_raw"):
    try: globals().pop(_n, None) and _.close()
    except: pass

# Configs...
# USER_VALUE_SPEC = kc.json_spec({...})
# ACCOUNT_VALUE_SPEC = kc.json_spec({...})

user_raw = kc.consume(KAFKA_CONFIG, TOPIC_USERS,  key_spec=kc.KeyValueSpec.IGNORE,
                      value_spec=USER_VALUE_SPEC,  table_type=kc.TableType.append)
account_raw = kc.consume(KAFKA_CONFIG, TOPIC_ACCTS, key_spec=kc.KeyValueSpec.IGNORE,
                         value_spec=ACCOUNT_VALUE_SPEC, table_type=kc.TableType.append)

user    = user_raw.rename_columns(["ts = KafkaTimestamp"])
account = account_raw.rename_columns(["ts = KafkaTimestamp"])
users_hist   = user.sort_descending("ts")
user_account = account.aj(users_hist, on=["userId","ts"], joins=["name","email","age"])
```

Re-running this cell will briefly swap the globals; the **hub** will see a single “closed” and reattach once. No flood.

---

## Why you were seeing “multiple attach … dh-core.js … ClassCastException”

- Each call to your old `bind()` opened **another** viewport and listeners.
    
- With Angular dev server/HMR, components can mount twice; with retries, you ended up with many viewports on the same table.
    
- When the Python cell re-ran, all those old viewers attempted to snapshot → multiple errors per tick.
    
- The hub removes that multiplicity; one viewport, one set of listeners, fan-out to any number of subscribers.
    

---

If anything still throws, check that **every** consumer component calls the returned `unsubscribe()` in `ngOnDestroy` (the example does). That, plus the hub, eliminates the attach storm and the “table already closed” spam.

--------------------------------

You’re seeing that storm because the frontend is opening **multiple viewports on the same DH global** (every bind/reattach from HMR or re-renders adds another), so when you re-run the Python cell and DH swaps the globals, all those stale viewports start firing `snapshot()` → `ClassCastException` → `Table already closed`.

Fix = make binding **single-flight per table variable** and **broadcast** updates to any listeners. That way there is only ever **one** viewport per DH global. Also make the binder self-healing (reattach once when the table is swapped) and make unsubscribe deterministic.

Below is an end-to-end update that addresses all the issues you’ve hit:

- PSK login (unchanged).
    
- A **TableHub** inside the service that:
    
    - de-duplicates by `varName`
        
    - keeps exactly **one** viewport open per table
        
    - snapshots once and **broadcasts** rows to all subscribers
        
    - self-heals on “closed” errors without flooding
        
    - reference-counts and closes the viewport when the last subscriber unsubscribes
        
- A minimal `LiveTableComponent` using the hub; it never calls `snapshot()` directly, so there’s no attach storm.
    
- Server side: keep your **top-level** DH cell (no helpers) that recreates `user`, `account`, `user_account` each run. The hub will reattach when that happens.
    

---

## 1) Deephaven service (PSK + TableHub)

`src/app/deephaven/deephaven.service.ts`

```ts
import { Injectable } from '@angular/core';
import { environment } from '../../environments/environment';

type DhNS = any;
type DhClient = any;
type DhIDE = any;
export type DhTable = any;

type RowsListener = (rows: any[]) => void;

interface HubEntry {
  varName: string;
  table?: DhTable;
  viewport?: any;
  cols: string[];
  listeners: Set<RowsListener>;
  refCount: number;
  generation: number;
  reconnecting: boolean;
  destroyed: boolean;
}

@Injectable({ providedIn: 'root' })
export class DeephavenService {
  private dh!: DhNS;
  private client!: DhClient;
  private ide!: DhIDE;
  private ready = false;

  getDhNs() { return this.dh; }
  get isReady() { return this.ready; }

  /** PSK login – same as before */
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

  async getTableHandle(varName: string): Promise<DhTable> {
    if (!this.ready) await this.connect();
    if (!/^[A-Za-z_][A-Za-z0-9_]*$/.test(varName)) {
      throw new Error(`Invalid table variable name: "${varName}"`);
    }
    return this.ide.getTable(varName);
  }

  // --------------------------- TableHub ---------------------------

  private hub = new Map<string, HubEntry>();

  /**
   * Subscribe to a DH globals() table by variable name.
   * Ensures ONLY ONE viewport per table variable.
   * Returns an unsubscribe() that decrements the ref and cleans up if last.
   */
  async subscribe(
    varName: string,
    onRows: RowsListener,
    opts?: { offset?: number; size?: number }
  ): Promise<() => Promise<void>> {
    await this.connect();
    let entry = this.hub.get(varName);
    if (!entry) {
      entry = {
        varName,
        cols: [],
        listeners: new Set<RowsListener>(),
        refCount: 0,
        generation: 0,
        reconnecting: false,
        destroyed: false,
      };
      this.hub.set(varName, entry);
      // Lazy attach on first subscriber
    }

    entry.refCount++;
    entry.listeners.add(onRows);

    if (!entry.table) {
      await this.attach(entry, opts);
    } else if (opts) {
      await this.setWindow(entry, opts.offset ?? 0, opts.size ?? 300, onRows);
    } else {
      // push a snapshot to the new listener immediately
      await this.pushSnapshot(entry, onRows).catch(() => {});
    }

    // Unsubscribe
    return async () => {
      if (!this.hub.has(varName)) return;
      const e = this.hub.get(varName)!;
      e.listeners.delete(onRows);
      e.refCount = Math.max(0, e.refCount - 1);
      if (e.refCount === 0) {
        await this.detach(e);
        this.hub.delete(varName);
      }
    };
  }

  // ---------- internals ----------

  private isClosed(e: any): boolean {
    const m = (e?.message ?? String(e)).toLowerCase();
    return m.includes('closed') || m.includes('table already closed');
  }

  private toRows(entry: HubEntry, objects: any[]): any[] {
    return objects.map(o => (Array.isArray(o) ? o : entry.cols.map(c => o?.[c])));
  }

  private eventNames() {
    const ns = this.getDhNs();
    return {
      EVU: ns?.Table?.EVENT_UPDATED ?? 'update',
      EVS: ns?.Table?.EVENT_SCHEMA_CHANGED ?? 'schema_changed',
    };
  }

  private async attach(entry: HubEntry, opts?: { offset?: number; size?: number }) {
    const { EVU, EVS } = this.eventNames();
    const myGen = ++entry.generation;
    const offset = opts?.offset ?? 0;
    const size = opts?.size ?? 300;

    // (Re)acquire table
    entry.table = await this.getTableHandle(entry.varName);
    entry.cols = (entry.table.columns ?? []).map((c: any) => c.name);

    // Open viewport
    entry.viewport = await entry.table.setViewport(offset, offset + size - 1, entry.cols);

    // First snapshot & broadcast
    await this.broadcastSnapshot(entry).catch(async (e) => {
      if (this.isClosed(e)) await this.reattach(entry);
    });

    // Bind listeners (single instance per varName)
    const onUpdate = async () => {
      if (entry.destroyed || myGen !== entry.generation) return;
      try {
        await this.broadcastSnapshot(entry);
      } catch (e) {
        if (this.isClosed(e)) await this.reattach(entry);
      }
    };
    const onSchema = async () => {
      if (entry.destroyed || myGen !== entry.generation) return;
      try {
        entry.cols = (entry.table!.columns ?? []).map((c: any) => c.name);
        await entry.table!.setViewport(offset, offset + size - 1, entry.cols);
        await this.broadcastSnapshot(entry);
      } catch (e) {
        if (this.isClosed(e)) await this.reattach(entry);
      }
    };

    // Store so we can remove later
    (entry as any)._onUpdate = onUpdate;
    (entry as any)._onSchema = onSchema;

    entry.table.addEventListener(EVU, onUpdate);
    entry.table.addEventListener(EVS, onSchema);
  }

  private async detach(entry: HubEntry) {
    entry.destroyed = true;
    try {
      const { EVU, EVS } = this.eventNames();
      if (entry.table && (entry as any)._onUpdate) {
        entry.table.removeEventListener?.(EVU, (entry as any)._onUpdate);
      }
      if (entry.table && (entry as any)._onSchema) {
        entry.table.removeEventListener?.(EVS, (entry as any)._onSchema);
      }
    } catch {}
    try { await entry.viewport?.close?.(); } catch {}
    entry.viewport = undefined;
    // Do NOT call table.close(); it's a server global.
    entry.table = undefined;
  }

  private async reattach(entry: HubEntry) {
    if (entry.reconnecting || entry.destroyed) return;
    entry.reconnecting = true;
    try {
      await this.detach(entry);
      await new Promise(r => setTimeout(r, 150)); // let DH finish swapping globals
      await this.attach(entry);
    } finally {
      entry.reconnecting = false;
    }
  }

  private async pushSnapshot(entry: HubEntry, toOne?: RowsListener) {
    if (!entry.table) return;
    const snap = await entry.table.snapshot(entry.cols);
    const objs = snap.toObjects?.() ?? snap;
    const rows = this.toRows(entry, objs);
    if (toOne) toOne(rows);
  }

  private async broadcastSnapshot(entry: HubEntry) {
    if (!entry.table || entry.listeners.size === 0) return;
    const snap = await entry.table.snapshot(entry.cols);
    const objs = snap.toObjects?.() ?? snap;
    const rows = this.toRows(entry, objs);
    entry.listeners.forEach(l => l(rows));
  }

  private async setWindow(entry: HubEntry, offset: number, size: number, toOne?: RowsListener) {
    if (!entry.table) return;
    await entry.table.setViewport(offset, offset + size - 1, entry.cols);
    if (toOne) await this.pushSnapshot(entry, toOne);
  }
}
```

**Why this stops the error flood**

- Each table variable (`'user'`, `'account'`, `'user_account'`) has **one** viewport and **one** set of listeners in the hub, regardless of how many components subscribe.
    
- On DH cell re-run, stale handles throw “closed” once → hub **reattaches once** and resumes.
    
- Late events from old generations are ignored.
    

---

## 2) Live component (simple consumer of the hub)

`src/app/live-table/live-table.component.ts`

```ts
import { Component, Input, OnInit, OnDestroy, ChangeDetectionStrategy, ChangeDetectorRef } from '@angular/core';
import { CommonModule } from '@angular/common';
import { DeephavenService } from '../deephaven/deephaven.service';

@Component({
  selector: 'app-live-table',
  standalone: true,
  imports: [CommonModule],
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <section class="card">
      <header class="bar">
        <h3><span class="mono">{{ varName }}</span></h3>
        <div *ngIf="error" class="error">{{ error }}</div>
      </header>

      <div class="grid" *ngIf="!error">
        <div class="thead" *ngIf="cols.length">
          <div class="th" *ngFor="let c of cols; trackBy: trackCol">{{ c }}</div>
        </div>
        <div class="tbody">
          <div class="tr" *ngFor="let r of rows; trackBy: trackRow">
            <div class="td" *ngFor="let v of r; trackBy: trackCell">{{ v }}</div>
          </div>
        </div>
      </div>
    </section>
  `,
  styles: [`
    :host{display:block}.card{border:1px solid #2a2a2a;border-radius:.75rem;background:#0f0f0f;color:#fff}
    .bar{display:flex;justify-content:space-between;align-items:center;padding:.75rem 1rem;border-bottom:1px solid #242424}
    .mono{font-family:ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, "Liberation Mono", monospace}
    .grid{display:grid;grid-template-rows:auto 1fr;gap:.5rem}
    .thead,.tr{display:grid;grid-auto-flow:column;grid-auto-columns:minmax(140px,1fr)}
    .th{position:sticky;top:0;background:#111;font-weight:600;padding:.5rem;z-index:1}
    .tbody{max-height:40dvh;overflow:auto}
    .td{padding:.25rem .5rem;border-top:1px solid #242424;white-space:nowrap;overflow:hidden;text-overflow:ellipsis}
    .error{color:#ff6b6b}
  `]
})
export class LiveTableComponent implements OnInit, OnDestroy {
  /** DH globals() variable name */
  @Input({ required: true }) varName!: string;
  @Input() pageSize = 300;

  cols: string[] = [];
  rows: any[] = [];
  error: string | null = null;

  private unsub?: () => Promise<void>;

  constructor(private dh: DeephavenService, private cdr: ChangeDetectorRef) {}

  async ngOnInit() {
    try {
      await this.dh.connect();
      // Pull columns once (optional)
      const t = await this.dh.getTableHandle(this.varName);
      this.cols = (t.columns ?? []).map((c: any) => c.name);

      // Subscribe via the hub (single viewport per table)
      this.unsub = await this.dh.subscribe(this.varName, rows => {
        this.rows = rows; this.cdr.markForCheck();
      }, { offset: 0, size: this.pageSize });
    } catch (e: any) {
      this.error = e?.message ?? String(e);
      this.cdr.markForCheck();
    }
  }

  async ngOnDestroy() { try { await this.unsub?.(); } catch {} }

  trackCol = (_: number, c: string) => c;
  trackRow = (_: number, r: any[]) => r?.[0] ?? _;
  trackCell = (_: number, _v: any) => _;
}
```

Use it three times in your page:

```html
<app-live-table varName="user"></app-live-table>
<app-live-table varName="account"></app-live-table>
<app-live-table varName="user_account"></app-live-table>
```

No ag-grid needed for this test; once stable, you can feed `rows`/`cols` into ag-grid as before – the hub makes the data flow identical (just remove any direct `snapshot()` calls from components).

---

## 3) Deephaven Python cell (top-level, unconditional)

Keep using the **single top-level** cell you now have (no helper functions, no `if in globals()` guards), e.g.:

```python
from deephaven.stream.kafka import consumer as kc
from deephaven import dtypes as dt

# Clean old names
for _n in ("user_account","users_hist","user","account","user_raw","account_raw"):
    try: globals().pop(_n, None) and _.close()
    except: pass

# Configs...
# USER_VALUE_SPEC = kc.json_spec({...})
# ACCOUNT_VALUE_SPEC = kc.json_spec({...})

user_raw = kc.consume(KAFKA_CONFIG, TOPIC_USERS,  key_spec=kc.KeyValueSpec.IGNORE,
                      value_spec=USER_VALUE_SPEC,  table_type=kc.TableType.append)
account_raw = kc.consume(KAFKA_CONFIG, TOPIC_ACCTS, key_spec=kc.KeyValueSpec.IGNORE,
                         value_spec=ACCOUNT_VALUE_SPEC, table_type=kc.TableType.append)

user    = user_raw.rename_columns(["ts = KafkaTimestamp"])
account = account_raw.rename_columns(["ts = KafkaTimestamp"])
users_hist   = user.sort_descending("ts")
user_account = account.aj(users_hist, on=["userId","ts"], joins=["name","email","age"])
```

Re-running this cell will briefly swap the globals; the **hub** will see a single “closed” and reattach once. No flood.

---

## Why you were seeing “multiple attach … dh-core.js … ClassCastException”

- Each call to your old `bind()` opened **another** viewport and listeners.
    
- With Angular dev server/HMR, components can mount twice; with retries, you ended up with many viewports on the same table.
    
- When the Python cell re-ran, all those old viewers attempted to snapshot → multiple errors per tick.
    
- The hub removes that multiplicity; one viewport, one set of listeners, fan-out to any number of subscribers.
    

---

If anything still throws, check that **every** consumer component calls the returned `unsubscribe()` in `ngOnDestroy` (the example does). That, plus the hub, eliminates the attach storm and the “table already closed” spam.

----------------------------

Got it — you’re still seeing a flood of `java.lang.ClassCastException` plus “Table already closed” even after pointing the JSAPI at the right DH origin. That pattern means the browser is still issuing **overlapping snapshots/viewports against a handle that just changed**. In practice, it happens when more than one place in the app is talking directly to the same DH table (or when HMR leaves a stale component alive), so your single-flight guard never “owns” the pipe.

Here’s how to make it stable end-to-end:

---

# 0) One way in, one way out (contract)

- ✅ **Only the service** touches Deephaven tables (getTable / setViewport / snapshot).
    
- ✅ Components **never** call `getTableHandle`, `snapshot`, or `setViewport` themselves.
    
- ✅ There is **exactly one subscription** per table var name in the whole page.
    
- ✅ On destroy, you **await** the unsubscriber so the viewport really closes.
    

If any of those are violated, you’ll get the exact exceptions you’re seeing.

---

# 1) Server cell: top-level, idempotent, no functions

Keep your DH Python like this (top level only). Re-running is safe; it just replaces the globals. Do not wrap in functions.

```python
from deephaven.stream.kafka import consumer as kc
from deephaven import dtypes as dt

TOPIC_USERS  = "ccd01_sb_its_esp_tap3567_bishowcasesraw"
TOPIC_ACCTS  = "ccd01_sb_its_esp_tap3567_bishowcasescurated"

KAFKA_CONFIG = {
    "bootstrap.servers": "pkc-...:9092",
    "security.protocol": "SASL_SSL",
    "sasl.mechanism": "OAUTHBEARER",
    # ... your working oauth configs ...
}

USER_VALUE_SPEC = kc.json_spec({
    "userId": dt.string, "name": dt.string, "email": dt.string, "age": dt.int64
})
ACCOUNT_VALUE_SPEC = kc.json_spec({
    "userId": dt.string, "accountType": dt.string, "balance": dt.double
})

# Consumers (append ticking tables)
user    = kc.consume(KAFKA_CONFIG, TOPIC_USERS, key_spec=kc.KeyValueSpec.IGNORE,
                     value_spec=USER_VALUE_SPEC, table_type=kc.TableType.append)
account = kc.consume(KAFKA_CONFIG, TOPIC_ACCTS, key_spec=kc.KeyValueSpec.IGNORE,
                     value_spec=ACCOUNT_VALUE_SPEC, table_type=kc.TableType.append)

# As-of join – row appears only when an account arrives; user fields as of that time
users_hist   = user.sort_descending("KafkaTimestamp")
user_account = account.aj(users_hist, on=["userId", "KafkaTimestamp"], joins=["name","email","age"])
```

Three live globals now exist and **stay**: `user`, `account`, `user_account`.

---

# 2) Service: Hub owns viewports & snapshots (single-flight + generation guard)

Use the service below (PSK login included). It ensures:

- one viewport per table var name,
    
- exactly one snapshot in flight,
    
- coalesced updates,
    
- automatic reattach if the server swaps the global.
    

```ts
// src/app/deephaven/deephaven.service.ts
import { Injectable } from '@angular/core';
import { environment } from '../../environments/environment';

type DhNS = any; type DhClient = any; type DhIDE = any; type DhTable = any;
type RowsListener = (rows: any[]) => void;

interface HubEntry {
  varName: string;
  table?: DhTable;
  viewport?: any;
  cols: string[];
  listeners: Set<RowsListener>;
  refCount: number;
  generation: number;
  destroyed: boolean;
  reconnecting: boolean;
  inFlight: Promise<void> | null;
  queued: boolean;
}

@Injectable({ providedIn: 'root' })
export class DeephavenService {
  private dh!: DhNS; private client!: DhClient; private ide!: DhIDE; private ready = false;

  get isReady() { return this.ready; }
  getDhNs() { return this.dh; }

  // --- PSK connect (hardened URL normalization) ---
  async connect(): Promise<void> {
    if (this.ready) return;
    const base = environment.deephavenUrl?.trim();
    const psk  = environment.deephavenPsk?.trim();
    if (!base) throw new Error('environment.deephavenUrl is not set');
    if (!psk)  throw new Error('environment.deephavenPsk is not set');

    const u = base.includes('://') ? new URL(base) : new URL(`http://${base}`);
    const serverUrl = `${u.protocol}//${u.hostname}${u.port ? ':' + u.port : ''}`;
    const jsapiUrl  = `${serverUrl}/jsapi/dh-core.js`;
    const mod = await import(/* @vite-ignore */ jsapiUrl);
    this.dh = mod.default;

    this.client = new this.dh.CoreClient(serverUrl);
    await this.client.login({ type: 'io.deephaven.authentication.psk.PskAuthenticationHandler', token: psk });
    this.ide = await this.client.getAsIdeConnection();
    await this.ide.startSession('python');
    this.ready = true;
  }

  async getTableHandle(varName: string): Promise<DhTable> {
    if (!this.ready) await this.connect();
    if (!/^[A-Za-z_][A-Za-z0-9_]*$/.test(varName)) throw new Error(`Bad var: ${varName}`);
    return this.ide.getTable(varName);
  }

  // ---------------- HUB ----------------
  private hub = new Map<string, HubEntry>();
  private ev() {
    const ns = this.getDhNs(); return {
      EVU: ns?.Table?.EVENT_UPDATED ?? 'update',
      EVS: ns?.Table?.EVENT_SCHEMA_CHANGED ?? 'schema_changed',
    };
  }
  private isClosed(e:any){ const m=(e?.message??String(e)).toLowerCase(); return m.includes('closed'); }
  private toRows(entry:HubEntry, objs:any[]){ return objs.map(o=>Array.isArray(o)?o:entry.cols.map(c=>o?.[c])); }

  async subscribe(varName:string, onRows:RowsListener, opts?:{offset?:number; size?:number}) {
    await this.connect();
    let e = this.hub.get(varName);
    if (!e) {
      e = { varName, cols:[], listeners:new Set(), refCount:0, generation:0, destroyed:false,
            reconnecting:false, inFlight:null, queued:false };
      this.hub.set(varName, e);
    }
    e.refCount++; e.listeners.add(onRows);
    if (!e.table) await this.attach(e, opts); else if (opts) await this.setWindow(e, opts.offset??0, opts.size??300, onRows);
    else await this.pushSnapshot(e, onRows).catch(()=>{});
    return async () => {
      const x = this.hub.get(varName); if (!x) return;
      x.listeners.delete(onRows); x.refCount=Math.max(0,x.refCount-1);
      if (x.refCount===0){ await this.detach(x); this.hub.delete(varName); }
    };
  }

  private async attach(e:HubEntry, opts?:{offset?:number; size?:number}){
    const {EVU,EVS}=this.ev(); const myGen=++e.generation; const off=opts?.offset??0; const size=opts?.size??300;
    e.destroyed=false;
    e.table = await this.getTableHandle(e.varName); if (e.destroyed||myGen!==e.generation) return;
    e.cols  = (e.table.columns??[]).map((c:any)=>c.name);
    e.viewport = await e.table.setViewport(off, off+size-1, e.cols); if (e.destroyed||myGen!==e.generation) return;

    // initial deliver
    await this.broadcastSnapshot(e).catch(async err=>{ if (this.isClosed(err)) await this.reattach(e); });

    const onUpd = () => { if (!e.destroyed) this.queueSnapshot(e); };
    const onSch = async () => {
      if (e.destroyed) return;
      try {
        const g=e.generation; e.cols=(e.table!.columns??[]).map((c:any)=>c.name);
        await e.table!.setViewport(off, off+size-1, e.cols);
        if (g!==e.generation||e.destroyed) return;
        this.queueSnapshot(e);
      } catch(err){ if (this.isClosed(err)) await this.reattach(e); }
    };
    (e as any)._onUpd=onUpd; (e as any)._onSch=onSch;
    e.table.addEventListener(EVU,onUpd); e.table.addEventListener(EVS,onSch);
  }

  private async detach(e:HubEntry){
    e.destroyed=true;
    try { const {EVU,EVS}=this.ev(); e.table?.removeEventListener?.(EVU,(e as any)._onUpd);
          e.table?.removeEventListener?.(EVS,(e as any)._onSch); } catch {}
    try { await e.viewport?.close?.(); } catch {}
    e.viewport=undefined; e.table=undefined; e.inFlight=null; e.queued=false;
  }

  private async reattach(e:HubEntry){
    if (e.reconnecting||e.destroyed) return; e.reconnecting=true;
    try { await this.detach(e); await new Promise(r=>setTimeout(r,150)); await this.attach(e); }
    finally { e.reconnecting=false; }
  }

  private async setWindow(e:HubEntry, off:number, size:number, toOne?:RowsListener){
    if (!e.table) return; const g=e.generation;
    await e.table.setViewport(off, off+size-1, e.cols);
    if (e.destroyed||g!==e.generation) return;
    if (toOne) await this.pushSnapshot(e,toOne);
  }

  // --- snapshots (single-flight + coalesce) ---
  private queueSnapshot(e:HubEntry){
    if (e.inFlight){ e.queued=true; return; }
    e.inFlight=(async()=>{
      const g=e.generation;
      try { await this.broadcastSnapshot(e); }
      catch(err){ if (this.isClosed(err)) await this.reattach(e); }
      finally{
        e.inFlight=null;
        if (e.queued && !e.destroyed && g===e.generation){ e.queued=false; this.queueSnapshot(e); }
      }
    })();
  }

  private async pushSnapshot(e:HubEntry, toOne:RowsListener){
    if (!e.table) return; const g=e.generation;
    const snap = await e.table.snapshot(e.cols); if (e.destroyed||g!==e.generation) return;
    const rows = this.toRows(e, snap.toObjects?.() ?? snap); toOne(rows);
  }
  private async broadcastSnapshot(e:HubEntry){
    if (!e.table || e.listeners.size===0) return; const g=e.generation;
    const snap = await e.table.snapshot(e.cols); if (e.destroyed||g!==e.generation) return;
    const rows = this.toRows(e, snap.toObjects?.() ?? snap);
    e.listeners.forEach(l=>l(rows));
  }
}
```

---

# 3) Component: subscribe once, unsubscribe once

A tiny “live table” view that **only** talks to the service. No DH calls in the component.

```ts
// src/app/live-table/live-table.component.ts
import { Component, Input, OnInit, OnDestroy } from '@angular/core';
import { DeephavenService } from '../deephaven/deephaven.service';

@Component({
  selector: 'app-live-table',
  standalone: true,
  template: `
  <section class="card">
    <header><h3>{{ varName }}</h3></header>
    <div class="thead">
      <div class="th" *ngFor="let c of cols">{{ c }}</div>
    </div>
    <div class="tbody">
      <div class="tr" *ngFor="let r of rows; trackBy: trackRow">
        <div class="td" *ngFor="let v of r; trackBy: trackCell">{{ v }}</div>
      </div>
    </div>
  </section>
  `,
})
export class LiveTableComponent implements OnInit, OnDestroy {
  @Input() varName!: string;
  rows: any[] = [];
  cols: string[] = [];
  private unsub: (() => Promise<void>) | null = null;

  constructor(private dh: DeephavenService) {}
  trackRow = (_: number, r: any[]) => r?.[0] ?? _;
  trackCell = (_: number) => _;

  async ngOnInit() {
    await this.dh.connect();
    // columns once (no snapshot overlap): read from getTableHandle
    const h = await this.dh.getTableHandle(this.varName);
    this.cols = (h.columns ?? []).map((c: any) => c.name);
    this.unsub = await this.dh.subscribe(this.varName, (rows) => { this.rows = rows; });
  }
  async ngOnDestroy() { try { await this.unsub?.(); } catch {} }
}
```

Use it three times:

```html
<app-live-table varName="user"></app-live-table>
<app-live-table varName="account"></app-live-table>
<app-live-table varName="user_account"></app-live-table>
```

---

# 4) Kill the usual sources of duplicates

- **Angular HMR** can leave old components alive → page shows one instance, but subscriptions from the previous instance still run. Do a **hard refresh** (Ctrl+Shift+R) after code edits while debugging this.
    
- Search the repo for any **other** calls to:
    
    - `.getTable(` outside the service
        
    - `.snapshot(` outside the service
        
    - `.setViewport(` outside the service  
        Remove them. One path only.
        
- Do not create your own `CoreClient` anywhere else.
    

---

## Why this stops your errors

- `ClassCastException` in DH snapshots is almost always the race “snapshot arrives for an object that’s not the current viewport/table any more.” The single-flight queue + generation checks ensure we **never apply** a stale snapshot.
    
- “Table already closed” happens when a listener tries to use an old handle. We detach listeners first, close the viewport, then replace the table, and only then start listening again. If the server swap happens mid-flight, we catch the “closed” and **reattach** once.
    

Follow the contract (one hub, one path, one subscription), drop these files in, and hard-refresh. If **any** component still runs its own snapshots, you’ll see the same stack. Remove that and it settles.

------------------------

You’ve done the right things (PSK login, single service, one subscription), but the stack you’re still seeing:

- `java.lang.ClassCastException`
    
- followed by “Table already closed; cannot be used again”
    

means the client is still calling the **server-side snapshot** endpoint while the table/viewport is being swapped underneath it. On some Deephaven builds the `snapshot()` RPC is brittle when the schema contains Kafka metadata (e.g., `KafkaOffset`/`KafkaTimestamp`), and if a rebalance or schema event lands mid-RPC you get that `ClassCastException`. Once the RPC fails, your code retries, but the old handle just got replaced → “table already closed”.

### The fix

Stop using `table.snapshot(...)` and read from the **viewport** only:

- open a viewport once,
    
- on every `EVENT_UPDATED` fetch the data **via the viewport** (`getViewportData()`),
    
- coalesce concurrent reads (single-flight),
    
- reattach if the table is swapped.
    

Below is a drop-in update for your service that replaces the snapshot logic. Nothing else in your app has to change. (You can keep your top-level Python that defines `user`, `account`, `user_account`.)

---

## Updated `DeephavenService` (PSK + viewport-only, no `snapshot()`)

```ts
// src/app/deephaven/deephaven.service.ts
import { Injectable } from '@angular/core';
import { environment } from '../../environments/environment';

type DhNS = any; type DhClient = any; type DhIDE = any; type DhTable = any;
type RowsListener = (rows: any[]) => void;

interface HubEntry {
  varName: string;
  table?: DhTable;
  viewport?: any;
  cols: string[];
  listeners: Set<RowsListener>;
  refCount: number;
  generation: number;
  destroyed: boolean;
  reconnecting: boolean;
  readInFlight: Promise<void> | null;
  readQueued: boolean;
}

@Injectable({ providedIn: 'root' })
export class DeephavenService {
  private dh!: DhNS; private client!: DhClient; private ide!: DhIDE; private ready = false;

  get isReady() { return this.ready; }
  getDhNs() { return this.dh; }

  // ---- PSK login (hardened) ----
  async connect(): Promise<void> {
    if (this.ready) return;
    const base = environment.deephavenUrl?.trim();
    const psk  = environment.deephavenPsk?.trim();
    if (!base) throw new Error('environment.deephavenUrl is not set');
    if (!psk)  throw new Error('environment.deephavenPsk is not set');

    const u = base.includes('://') ? new URL(base) : new URL(`http://${base}`);
    const origin = `${u.protocol}//${u.hostname}${u.port ? ':' + u.port : ''}`;

    const jsapiUrl = `${origin}/jsapi/dh-core.js`;
    const mod = await import(/* @vite-ignore */ jsapiUrl);
    this.dh = mod.default;

    this.client = new this.dh.CoreClient(origin);
    await this.client.login({
      type: 'io.deephaven.authentication.psk.PskAuthenticationHandler',
      token: psk,
    });

    this.ide = await this.client.getAsIdeConnection();
    await this.ide.startSession('python');
    this.ready = true;
  }

  async getTableHandle(varName: string): Promise<DhTable> {
    if (!this.ready) await this.connect();
    if (!/^[A-Za-z_][A-Za-z0-9_]*$/.test(varName)) throw new Error(`Bad var: ${varName}`);
    return this.ide.getTable(varName);
  }

  // ---------------- HUB (viewport-only) ----------------
  private hub = new Map<string, HubEntry>();
  private ev() {
    const ns = this.getDhNs(); return {
      EVU: ns?.Table?.EVENT_UPDATED ?? 'update',
      EVS: ns?.Table?.EVENT_SCHEMA_CHANGED ?? 'schema_changed',
    };
  }
  private isClosed(e:any){ const m=(e?.message??String(e)).toLowerCase(); return m.includes('closed'); }

  async subscribe(varName:string, onRows:RowsListener, opts?:{offset?:number; size?:number}) {
    await this.connect();
    let e = this.hub.get(varName);
    if (!e) {
      e = { varName, cols:[], listeners:new Set(), refCount:0, generation:0, destroyed:false,
            reconnecting:false, readInFlight:null, readQueued:false };
      this.hub.set(varName, e);
    }
    e.refCount++; e.listeners.add(onRows);
    if (!e.table) await this.attach(e, opts);
    else if (opts) await this.setWindow(e, opts.offset ?? 0, opts.size ?? 300, onRows);
    else await this.readViewportAndBroadcast(e).catch(()=>{});
    return async () => {
      const x = this.hub.get(varName); if (!x) return;
      x.listeners.delete(onRows); x.refCount=Math.max(0,x.refCount-1);
      if (x.refCount===0){ await this.detach(x); this.hub.delete(varName); }
    };
  }

  private async attach(e:HubEntry, opts?:{offset?:number; size?:number}) {
    const {EVU,EVS}=this.ev(); const myGen=++e.generation;
    const off = opts?.offset ?? 0; const size = opts?.size ?? 300;

    e.destroyed=false;
    e.table = await this.getTableHandle(e.varName); if (e.destroyed || myGen!==e.generation) return;
    e.cols  = (e.table.columns ?? []).map((c:any) => c.name);

    // open a viewport (no snapshot)
    e.viewport = await e.table.setViewport(off, off+size-1, e.cols);
    if (e.destroyed || myGen!==e.generation) return;

    // first paint via viewport
    await this.readViewportAndBroadcast(e).catch(async err => { if (this.isClosed(err)) await this.reattach(e); });

    const onUpd = () => { if (!e.destroyed) this.queueViewportRead(e); };
    const onSch = async () => {
      if (e.destroyed) return;
      try {
        const g=e.generation;
        e.cols=(e.table!.columns??[]).map((c:any)=>c.name);
        await e.table!.setViewport(off, off+size-1, e.cols);
        if (g!==e.generation||e.destroyed) return;
        this.queueViewportRead(e);
      } catch(err){ if (this.isClosed(err)) await this.reattach(e); }
    };
    (e as any)._onUpd = onUpd; (e as any)._onSch = onSch;
    e.table.addEventListener(EVU, onUpd);
    e.table.addEventListener(EVS, onSch);
  }

  private async detach(e:HubEntry){
    e.destroyed=true;
    try { const {EVU,EVS}=this.ev(); e.table?.removeEventListener?.(EVU,(e as any)._onUpd);
          e.table?.removeEventListener?.(EVS,(e as any)._onSch); } catch {}
    try { await e.viewport?.close?.(); } catch {}
    e.viewport=undefined; e.table=undefined; e.readInFlight=null; e.readQueued=false;
  }

  private async reattach(e:HubEntry){
    if (e.reconnecting||e.destroyed) return; e.reconnecting=true;
    try { await this.detach(e); await new Promise(r=>setTimeout(r,150)); await this.attach(e); }
    finally { e.reconnecting=false; }
  }

  private async setWindow(e:HubEntry, off:number, size:number, toOne?:RowsListener){
    if (!e.table) return; const g=e.generation;
    await e.table.setViewport(off, off+size-1, e.cols);
    if (e.destroyed||g!==e.generation) return;
    if (toOne) await this.readViewportAndSendTo(e, toOne);
  }

  // ---- viewport read (single-flight + coalesce) ----
  private queueViewportRead(e:HubEntry){
    if (e.readInFlight){ e.readQueued=true; return; }
    e.readInFlight = (async ()=>{
      const g=e.generation;
      try { await this.readViewportAndBroadcast(e); }
      catch(err){ if (this.isClosed(err)) await this.reattach(e); }
      finally{
        e.readInFlight = null;
        if (e.readQueued && !e.destroyed && g===e.generation){ e.readQueued=false; this.queueViewportRead(e); }
      }
    })();
  }

  private async readViewportAndSendTo(e:HubEntry, toOne:RowsListener){
    if (!e.table) return; const g=e.generation;
    // IMPORTANT: use the viewport accessor, not table.snapshot()
    const data = await e.table.getViewportData();  // { columns, rows }
    if (e.destroyed || g!==e.generation) return;
    toOne(data?.rows ?? []);
  }
  private async readViewportAndBroadcast(e:HubEntry){
    if (!e.table || e.listeners.size===0) return; const g=e.generation;
    const data = await e.table.getViewportData();  // { columns, rows }
    if (e.destroyed || g!==e.generation) return;
    const rows = data?.rows ?? [];
    e.listeners.forEach(l=>l(rows));
  }
}
```

**What changed vs your last build**

- No calls to `table.snapshot(...)` anymore.
    
- All reads come through `table.getViewportData()`, which is designed for the current viewport window and doesn’t hit the server’s snapshot path that was throwing `ClassCastException`.
    
- Still single-flight (no overlapping reads), coalesced, and with reattach on “closed”.
    

---

## Component stays simple

Your `LiveTableComponent` can stay as you wrote it (subscribe once, unsubscribe once). If you want a minimal version:

```ts
import { Component, Input, OnInit, OnDestroy } from '@angular/core';
import { DeephavenService } from '../deephaven/deephaven.service';

@Component({
  selector: 'app-live-table',
  standalone: true,
  template: `
    <section>
      <h3>{{ varName }}</h3>
      <div class="thead"><span *ngFor="let c of cols">{{ c }}</span></div>
      <div class="tbody">
        <div class="tr" *ngFor="let r of rows; trackBy: trackRow">
          <span class="td" *ngFor="let v of r; trackBy: trackCell">{{ v }}</span>
        </div>
      </div>
    </section>
  `,
})
export class LiveTableComponent implements OnInit, OnDestroy {
  @Input() varName!: string;
  cols: string[] = []; rows: any[] = [];
  private unsub: (()=>Promise<void>)|null = null;

  constructor(private dh: DeephavenService) {}
  trackRow = (_: number, r: any[]) => r?.[0] ?? _;
  trackCell = (_: number) => _;

  async ngOnInit() {
    await this.dh.connect();
    const h = await this.dh.getTableHandle(this.varName);
    this.cols = (h.columns ?? []).map((c:any)=>c.name);
    this.unsub = await this.dh.subscribe(this.varName, rows => this.rows = rows);
  }
  async ngOnDestroy(){ try{ await this.unsub?.(); }catch{} }
}
```

Use it for all three tables:

```html
<app-live-table varName="user"></app-live-table>
<app-live-table varName="account"></app-live-table>
<app-live-table varName="user_account"></app-live-table>
```

---

## If you still see errors

1. **Hard refresh** the page after code edits (disable HMR while testing). Old instances can still be subscribed.
    
2. Ensure **no other code** calls `snapshot()` or `setViewport()` directly. The service must be the **only** place doing that.
    
3. If your DH version doesn’t yet expose `getViewportData()` (rare on older builds), switch to:
    
    ```ts
    const vp = e.viewport ?? await e.table!.setViewport(...);
    const data = await vp.getData();            // equivalent in some builds
    const rows = data?.rows ?? [];
    ```
    
4. If your joined table has 64-bit types that your UI doesn’t need, you can avoid any conversion weirdness by casting on the server:
    
    ```python
    user_account = user_account.update_view([
        "KafkaOffsetStr = (String)KafkaOffset",
        "KafkaTimestampStr = (String)KafkaTimestamp"
    ])
    ```
    
    and list those `...Str` columns in the UI.
    

This change (viewport-only reads) is the reliable workaround teams use in production to eliminate the `ClassCastException`/“closed” storm you’re hitting.