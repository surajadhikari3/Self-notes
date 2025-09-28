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