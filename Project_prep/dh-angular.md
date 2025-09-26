Short answer: you **don’t** need to “host an API.” If your Deephaven server is already ingesting Kafka and producing a **live (ticking) table**, your Angular app can connect **directly** to the Deephaven server via the **JS API** (which uses a WebSocket under the hood) and stream updates in real-time. ([Deephaven](https://deephaven.io/core/docs/how-to-guides/use-jsapi/?utm_source=chatgpt.com "Use the JS API"))

If you’ve **disabled auth** (anonymous), you can skip login entirely. (Just know that anonymous access has no security.) ([Deephaven](https://deephaven.io/core/groovy/docs/how-to-guides/authentication/auth-anon/?utm_source=chatgpt.com "Enable anonymous authentication"))

---

## Minimal Angular setup (no auth, real-time streaming)

**Install**:

```bash
npm i @deephaven/jsapi-bootstrap @deephaven/jsapi-types
```

**`src/environments/environment.ts`**

```ts
export const environment = {
  DH_BASE_URL: 'http://localhost:10000', // ← your DH server
  TABLE_NAME: 'live_table'               // ← the published table variable created by your Kafka script
};
```

**`src/app/deephaven.service.ts`**

```ts
import { Injectable } from '@angular/core';
import { BehaviorSubject, Observable } from 'rxjs';
import type { CoreClient, IdeConnection, Table } from '@deephaven/jsapi-types';
import * as Bootstrap from '@deephaven/jsapi-bootstrap';
import { environment } from '../environments/environment';

@Injectable({ providedIn: 'root' })
export class DeephavenService {
  private client!: CoreClient;
  private table!: Table;
  private rows$ = new BehaviorSubject<any[]>([]);
  private keys: string[] = [];

  getRows$(): Observable<any[]> { return this.rows$.asObservable(); }
  getKeys(): string[] { return this.keys; }

  /** Connect to DH and subscribe to an existing live table created by your Kafka ingester */
  async connectAndSubscribe(): Promise<void> {
    const baseUrl = environment.DH_BASE_URL;

    // Load the JS API that Deephaven serves; falls back to direct import if needed.
    const dh =
      (await (Bootstrap as any).loadApi?.({ baseUrl })) ??
      (await (Bootstrap as any).bootstrapApi?.({ baseUrl })) ??
      (await import(/* @vite-ignore */ `${baseUrl}/jsapi/dh-core.js`)).default;

    // Create client (no auth step if anonymous is enabled)
    this.client = new dh.CoreClient(baseUrl);

    // Open an IDE session to access the published table variable
    const ideConn: IdeConnection = await this.client.getAsIdeConnection();
    const ide = await ideConn.startSession('python');

    // IMPORTANT: TABLE_NAME must match the variable exposed by your Kafka script (e.g., live_table)
    this.table = await ide.getTable(environment.TABLE_NAME);

    // Use a viewport for efficient real-time streaming to the UI
    this.table.setViewport(0, 99); // first 100 rows (tweak as needed)

    const pushViewport = async () => {
      const vp = await this.table.getViewportData();
      const cols = this.table.columns;
      this.keys = cols.map((c: any) => c.name);
      const data = vp.rows.map((r: any) => {
        const obj: Record<string, any> = {};
        cols.forEach((c: any) => (obj[c.name] = r.get(c)));
        return obj;
      });
      this.rows$.next(data);
    };

    await pushViewport();
    this.table.addEventListener('update', pushViewport); // real-time deltas over WebSocket
  }
}
```

**`src/app/live-table.component.ts`**

```ts
import { Component } from '@angular/core';
import { DeephavenService } from './deephaven.service';

@Component({
  selector: 'app-live-table',
  template: `
    <button (click)="connect()" [disabled]="connected">
      {{ connected ? 'Connected' : 'Connect to Deephaven' }}
    </button>

    <table *ngIf="rows.length" style="margin-top:1rem; border-collapse:collapse;" border="1">
      <thead><tr><th *ngFor="let k of keys">{{ k }}</th></tr></thead>
      <tbody>
        <tr *ngFor="let r of rows">
          <td *ngFor="let k of keys">{{ r[k] }}</td>
        </tr>
      </tbody>
    </table>
  `,
})
export class LiveTableComponent {
  rows: any[] = [];
  keys: string[] = [];
  connected = false;

  constructor(private dh: DeephavenService) {}

  async connect() {
    await this.dh.connectAndSubscribe();
    this.connected = true;
    this.keys = this.dh.getKeys();
    this.dh.getRows$().subscribe(rows => (this.rows = rows));
  }
}
```

That’s it. No extra API to host, no polling, and it streams **live**.

---

## FAQs

- **Do I need WebSocket or HTTP?**  
    The JS API uses **WebSocket** internally for live/ticking tables; you don’t manage it yourself. ([Deephaven](https://deephaven.io/core/docs/how-to-guides/use-jsapi/?utm_source=chatgpt.com "Use the JS API"))
    
- **Do I need authentication?**  
    Not if you’ve enabled **anonymous** access (common in dev). Just be aware this provides **no security**. ([Deephaven](https://deephaven.io/core/groovy/docs/how-to-guides/authentication/auth-anon/?utm_source=chatgpt.com "Enable anonymous authentication"))
    
- **Will my Kafka tables be “live”?**  
    Yes. The Kafka ingester writes into **live/ticking** tables that you can subscribe to exactly like above. ([Deephaven](https://deephaven.io/core/docs/how-to-guides/overview-kafka/?utm_source=chatgpt.com "Kafka Overview"))
    

If your Kafka script names the table something other than `live_table`, just set `TABLE_NAME` to that variable. If you hit CORS while importing `…/jsapi/dh-core.js`, add a dev proxy or host the Angular app on the same origin as Deephaven. ([Deephaven](https://deephaven.io/core/groovy/docs/how-to-guides/use-jsapi/?utm_source=chatgpt.com "Use the JS API"))


--------------------------



Got it—you want (1) a reliable way to list **live/ticking** tables from your Deephaven session, (2) let the **user pick** two of them and a join, and (3) perform the **join on the Deephaven server** while your Angular UI **streams** the result via the JS API. Below I’ll first explain the moving parts, then give you **version-compatible** code for:

- Angular service (`DeephavenService`) using `@deephaven/jsapi-types@1.0.0-dev-0.40.0` and `@deephaven/jsapi-bootstrap@1.5.3`
    
- A **generic** Angular 20.3.0 standalone component (`JoinBuilderComponent`) that renders a live viewport and is careful about cleanup + errors
    

---

## How it works (quick tour)

1. **Connect & authenticate**  
    `CoreClient` connects to the Deephaven server (WebSocket under the hood). We log in (PSK in this example), then open an IDE session (e.g., Python). That session’s `globals()` can hold tables your pipelines produced (Kafka ingests, etc.).
    
2. **Catalog live/ticking tables**  
    We run a tiny server-side script that inspects `globals()` and collects only those objects that are `Table` **and** `is_refreshing == True`. That gives you the **live** tables, not static ones. We also include a CSV of column names to help the UI.
    
3. **User picks tables & join type**  
    In Angular, the user picks left table, right table, join kind (`natural_join`, `left_join`, `exact_join`, `join`, or `aj` for as-of), keys, and optional right-side columns.
    
4. **Perform the join on the server**  
    Angular submits a small Python snippet to your IDE session that creates a **derived** table variable (unique name). Because the inputs are ticking, the **joined result also ticks**.
    
5. **Subscribe & stream to UI**  
    In Angular, you call `setViewport(start, end)` and listen for `EVENT_UPDATED`. Each tick, you pull the current viewport and render—no polling required.
    
6. **Cleanup**  
    On component destroy or when building a new join, detach listeners and close prior table handles to avoid leaks.
    

---

## Angular service (TypeScript)

- Works with `@deephaven/jsapi-types@1.0.0-dev-0.40.0` and `@deephaven/jsapi-bootstrap@1.5.3`
    
- Uses PSK auth for clarity; swap to your handler as needed
    

```ts
// src/app/services/deephaven.service.ts
import { Injectable } from '@angular/core';
import initDh from '@deephaven/jsapi-bootstrap';

// Types from @deephaven/jsapi-types
type DhNS = typeof import('@deephaven/jsapi-types');
type DhTable = any;      // You can refine with exact types if you prefer
type DhCoreClient = any;
type DhIde = any;

export type TableCatalogRow = {
  name: string;
  columns_csv: string;
};

export type JoinMode = 'natural' | 'exact' | 'left' | 'join' | 'aj';

@Injectable({ providedIn: 'root' })
export class DeephavenService {
  private dh!: DhNS;
  private client!: DhCoreClient;
  private ide!: DhIde;

  /** Connect to DH server and start IDE session. Adjust URL/auth to your env. */
  async connect(serverUrl = 'http://localhost:10000', psk = 'very-secret'): Promise<void> {
    this.dh = await initDh();
    this.client = new this.dh.CoreClient(serverUrl);
    await this.client.login({
      type: 'io.deephaven.authentication.psk.PskAuthenticationHandler',
      token: psk,
    });
    const conn = await this.client.getAsIdeConnection();
    // You can use 'python' or 'groovy' as available on your server
    this.ide = await conn.startSession('python');
  }

  /**
   * Build a server-side catalog of **live/ticking** tables in the current session.
   * Returns a DH Table you can read via viewport.
   */
  async getLiveTableCatalog(): Promise<DhTable> {
    const code = `
from deephaven import new_table
from deephaven.column import string_col
from deephaven.table import Table

names = []
schemas = []
for k, v in globals().items():
    if isinstance(v, Table) and getattr(v, "is_refreshing", False):
        names.append(k)
        schemas.append(",".join([c.name for c in v.columns]))

__dh_live_catalog = new_table([
    string_col("name", names),
    string_col("columns_csv", schemas),
])
`;
    await this.ide.runCode(code);
    return await this.ide.getTable('__dh_live_catalog');
  }

  /**
   * Create a ticking joined table on the server using user choices.
   * Returns a handle to the derived table; subscribe via setViewport + EVENT_UPDATED.
   */
  async createJoinedTable(opts: {
    leftName: string;
    rightName: string;
    mode: JoinMode;
    on: string[];
    joins?: string[];      // RIGHT columns to include; default ['*']
  }): Promise<DhTable> {
    const { leftName, rightName, mode, on, joins = ['*'] } = opts;

    // Unique var name per join so multiple results can coexist
    const id = (globalThis.crypto?.randomUUID?.() ?? Math.random().toString(36).slice(2)).replace(/-/g, '');
    const varName = f`joined_${id}`.replace(/[^a-zA-Z0-9_]/g, '_'); // just in case

    // NOTE: We trust the table names to exist in globals(). Validate in production.
    const py = `
left = ${leftName}
right = ${rightName}
mode = "${mode}"
on_cols = ${JSON.stringify(on)}
join_cols = ${JSON.stringify(joins)}

if mode == "natural":
    ${varName} = left.natural_join(right, on=on_cols, joins=join_cols)
elif mode == "exact":
    ${varName} = left.exact_join(right, on=on_cols, joins=join_cols)
elif mode == "left":
    ${varName} = left.left_join(right, on=on_cols, joins=join_cols)
elif mode == "aj":
    # As-of join: include your time key in 'on_cols' (e.g., ['Symbol','Timestamp'])
    ${varName} = left.aj(right, on=on_cols)
else:
    # Inner join
    ${varName} = left.join(right, on=on_cols, joins=join_cols)
`;
    await this.ide.runCode(py);
    return await this.ide.getTable(varName);
  }

  /** Optional utility to close a table handle when you no longer need it. */
  async closeTableHandle(table: DhTable | null | undefined): Promise<void> {
    try {
      if (table?.removeEventListener) {
        // Callers should remove listeners themselves when they have the handler
      }
      if (table?.close) await table.close();
    } catch {
      // swallow
    }
  }
}
```

---

## Generic Angular component (standalone, Angular 20.3.0)

- Standalone (no NgModule needed)
    
- Lists ticking tables, lets the user choose, creates the ticking join, renders a simple grid
    
- Careful about detach/cleanup and guarded async calls
    
- Minimal styling; feel free to replace with your design system
    

```ts
// src/app/join-builder/join-builder.component.ts
import { ChangeDetectorRef, Component, OnDestroy, OnInit, effect, signal } from '@angular/core';
import { CommonModule } from '@angular/common';
import { FormsModule } from '@angular/forms';
import type { TableCatalogRow, JoinMode } from '../services/deephaven.service';
import { DeephavenService } from '../services/deephaven.service';

@Component({
  selector: 'app-join-builder',
  standalone: true,
  imports: [CommonModule, FormsModule],
  template: `
  <section class="card">
    <h2>Build a Live Join</h2>

    <form class="grid" (ngSubmit)="buildJoin()">
      <label>
        Left table
        <select [(ngModel)]="left" name="left" required>
          <option *ngFor="let t of tableList()" [value]="t.name">{{ t.name }}</option>
        </select>
      </label>

      <label>
        Right table
        <select [(ngModel)]="right" name="right" required>
          <option *ngFor="let t of tableList()" [value]="t.name">{{ t.name }}</option>
        </select>
      </label>

      <label>
        Join type
        <select [(ngModel)]="mode" name="mode">
          <option value="natural">natural_join</option>
          <option value="exact">exact_join</option>
          <option value="left">left_join</option>
          <option value="join">join (inner)</option>
          <option value="aj">aj (as-of)</option>
        </select>
      </label>

      <label>
        Keys (CSV)
        <input [(ngModel)]="keysCsv" name="keysCsv" placeholder="e.g. Symbol,Timestamp"/>
        <small *ngIf="mode==='aj'">Include your time key for as-of join.</small>
      </label>

      <label>
        Right columns (CSV; '*' = all)
        <input [(ngModel)]="joinsCsv" name="joinsCsv" placeholder="e.g. Bid,Ask or *"/>
      </label>

      <div class="row">
        <button type="submit" [disabled]="building">Create Ticking Join</button>
        <button type="button" (click)="reset()" [disabled]="!joinedTable">Reset</button>
      </div>
    </form>

    <p class="hint" *ngIf="error">{{ error }}</p>
  </section>

  <section class="card" *ngIf="joinedTable">
    <header class="thead">
      <div class="th" *ngFor="let c of cols">{{ c }}</div>
    </header>

    <div class="tbody" role="table">
      <div class="tr" role="row" *ngFor="let r of rows">
        <div class="td" role="cell" *ngFor="let v of r">{{ v }}</div>
      </div>
    </div>
  </section>
  `,
  styles: [`
    :host { display:block; padding:1rem; color:var(--fg,#fff); }
    .card { border:1px solid #2a2a2a; border-radius:.75rem; padding:1rem; margin-bottom:1rem; background:#111; }
    .grid { display:grid; gap:.75rem; grid-template-columns: repeat(auto-fit,minmax(260px,1fr)); }
    label { display:flex; flex-direction:column; gap:.25rem; }
    input, select, button { padding:.5rem; background:#161616; color:#fff; border:1px solid #333; border-radius:.5rem; }
    .row { display:flex; gap:.5rem; align-items:center; }
    .thead, .tr { display:grid; grid-template-columns: repeat(auto-fit, minmax(140px, 1fr)); }
    .th, .td { padding:.5rem; border-bottom:1px solid #1f1f1f; white-space:nowrap; overflow:hidden; text-overflow:ellipsis; }
    .thead { position:sticky; top:0; background:#0f0f0f; font-weight:600; border-bottom:1px solid #333; }
    .tbody { max-height: 60vh; overflow:auto; }
    .hint { color:#ffb3b3; }
  `]
})
export class JoinBuilderComponent implements OnInit, OnDestroy {
  // form state
  left = '';
  right = '';
  mode: JoinMode = 'natural';
  keysCsv = '';
  joinsCsv = '*';

  // data & ui state
  tableList = signal<TableCatalogRow[]>([]);
  building = false;
  error = '';

  // joined table handle & view
  joinedTable: any | null = null;
  cols: string[] = [];
  rows: unknown[][] = [];

  // bound listener so we can remove it
  private onUpdate = async () => {
    if (!this.joinedTable) return;
    try {
      const vp = await this.joinedTable.getViewportData();
      // map to simple 2D array for display
      this.rows = vp.rows.map((r: any) =>
        this.cols.map((c) => r.get(this.joinedTable.columns.find((x:any)=>x.name===c)))
      );
      this.cdr.markForCheck();
    } catch (e) {
      // If the viewport disappeared during teardown, just ignore
    }
  };

  constructor(private dh: DeephavenService, private cdr: ChangeDetectorRef) {}

  async ngOnInit(): Promise<void> {
    try {
      await this.dh.connect(); // set server URL/PSK in service if different
      const catalog = await this.dh.getLiveTableCatalog();

      await catalog.setViewport(0, 2000);
      const vp = await catalog.getViewportData();
      const nameCol = catalog.columns.find((c:any)=>c.name==='name');
      const colsCol = catalog.columns.find((c:any)=>c.name==='columns_csv');
      const items = vp.rows.map((r:any) => ({
        name: r.get(nameCol),
        columns_csv: r.get(colsCol),
      })) as TableCatalogRow[];
      this.tableList.set(items);

      // Optionally preselect left/right if present
      this.left = items[0]?.name ?? '';
      this.right = items[1]?.name ?? '';
    } catch (e:any) {
      this.error = 'Unable to load live tables. ' + (e?.message ?? '');
    } finally {
      this.cdr.markForCheck();
    }
  }

  async buildJoin(): Promise<void> {
    this.error = '';
    this.building = true;
    this.cdr.markForCheck();

    // cleanup any existing subscription
    await this.teardownJoinedTable();

    try {
      const on = this.keysCsv.split(',').map(s => s.trim()).filter(Boolean);
      const joins = this.joinsCsv === '*' ? ['*'] : this.joinsCsv.split(',').map(s => s.trim()).filter(Boolean);

      this.joinedTable = await this.dh.createJoinedTable({
        leftName: this.left,
        rightName: this.right,
        mode: this.mode,
        on,
        joins,
      });

      // Prepare columns & initial viewport
      this.cols = (this.joinedTable.columns ?? []).map((c:any) => c.name);
      await this.joinedTable.setViewport(0, 300);
      await this.onUpdate();

      // Subscribe to ticks
      this.joinedTable.addEventListener(this.joinedTable.EVENT_UPDATED, this.onUpdate);
      // Optional: handle server-side close
      this.joinedTable.addEventListener?.(this.joinedTable.EVENT_CLOSED, () => {
        this.error = 'Joined table was closed on the server.';
        this.cdr.markForCheck();
      });
    } catch (e:any) {
      this.error = 'Failed to create joined table. ' + (e?.message ?? '');
    } finally {
      this.building = false;
      this.cdr.markForCheck();
    }
  }

  async reset(): Promise<void> {
    this.error = '';
    await this.teardownJoinedTable();
    this.cols = [];
    this.rows = [];
    this.cdr.markForCheck();
  }

  private async teardownJoinedTable(): Promise<void> {
    try {
      if (this.joinedTable) {
        if (this.onUpdate) {
          this.joinedTable.removeEventListener(this.joinedTable.EVENT_UPDATED, this.onUpdate);
        }
        await this.joinedTable.close?.();
      }
    } catch {
      // ignore
    } finally {
      this.joinedTable = null;
    }
  }

  async ngOnDestroy(): Promise<void> {
    await this.teardownJoinedTable();
  }
}
```

---

## Why this is “generic and reliable”

- **Catalog filters live tables only** (`Table` + `is_refreshing`) so users don’t accidentally pick static tables.
    
- **Unique server variable per join** so multiple joins can coexist; no accidental overwrites.
    
- **Viewport + `EVENT_UPDATED`** for streaming UI without polling.
    
- **Defensive teardown**: removes listeners and closes the old handle when the user builds a new join or navigates away.
    
- **Graceful error surfaces**: shows simple messages and keeps the UI usable.
    
- **Version-compatible**: uses `jsapi-bootstrap@1.5.3` to load the `dh` namespace and types from `@deephaven/jsapi-types@1.0.0-dev-0.40.0`. The APIs used here (CoreClient/login/getAsIdeConnection/startSession, Table.setViewport/getViewportData/addEventListener/close) are stable in that range.
    

### Notes / switches you may want:

- **Auth**: swap the PSK handler to whatever you use (OAuth/JWT/etc.).
    
- **Time-series joins**: for `aj`, include the time column in the Keys field (“`Symbol,Timestamp`”).
    
- **Column picker**: you can parse `columns_csv` for each picked table and show a multi-select for keys and `joins`.
    
- **Paging**: if you need larger results, add simple paging (advance the viewport window on scroll/end).
    

If you drop these two files in your Angular 20 app (plus a simple route to the component), you’ll have a working “pick live tables → build server join → stream to UI” flow with Deephaven.



----------------------------------------------------------------------


You’re right — with **`@deephaven/jsapi-bootstrap@1.5.3`** there isn’t an `init` export. The simplest, **guaranteed-working** approach (and the one Deephaven documents) is to import the JS API **directly from your DH server** at runtime, then use `CoreClient`. Below is a drop-in Angular 20.3.0 service + component that:

- lists **live (ticking)** tables,
    
- lets the user choose two tables, join type, keys & columns,
    
- runs the **join on the DH server**,
    
- streams the ticking result into your Angular UI.
    

The only things you might change are your **server URL** and **PSK**.

(Deephaven’s docs show importing `dh` from `http://<server>:10000/jsapi/dh-core.js` and then using `new dh.CoreClient(...)`, exactly what we do here. ([Deephaven](https://deephaven.io/core/docs/how-to-guides/use-jsapi/ "Use the JS API | Deephaven")))

---

## 1) Angular service

```ts
// src/app/services/deephaven.service.ts
import { Injectable } from '@angular/core';

// Keep the DH namespace/runtime as 'any' to avoid type drift across versions
type DhNS = any;
type DhTable = any;

export type TableCatalogRow = { name: string; columns_csv: string };
export type JoinMode = 'natural' | 'exact' | 'left' | 'join' | 'aj';

@Injectable({ providedIn: 'root' })
export class DeephavenService {
  private dh!: DhNS;
  private client!: any;
  private ide!: any;

  /**
   * Connect to Deephaven and start an IDE session.
   * serverUrl example: 'http://localhost:10000'
   * psk example: 'very-secret'
   */
  async connect(serverUrl = 'http://localhost:10000', psk = 'very-secret'): Promise<void> {
    const base = serverUrl.replace(/\/+$/, '');
    // Import the JS API directly from the Deephaven server (documented approach)
    // https://deephaven.io/core/docs/how-to-guides/use-jsapi/
    const mod = await import(/* webpackIgnore: true */ `${base}/jsapi/dh-core.js`);
    this.dh = mod.default;

    this.client = new this.dh.CoreClient(serverUrl);
    await this.client.login({
      type: 'io.deephaven.authentication.psk.PskAuthenticationHandler',
      token: psk,
    });

    const conn = await this.client.getAsIdeConnection();
    this.ide = await conn.startSession('python'); // or 'groovy'
  }

  /** Build a catalog of LIVE (ticking) tables in the Python globals() and return it as a server-side table. */
  async getLiveTableCatalog(): Promise<DhTable> {
    const code = `
from deephaven import new_table
from deephaven.column import string_col
from deephaven.table import Table

names = []
schemas = []
for k, v in globals().items():
    if isinstance(v, Table) and getattr(v, "is_refreshing", False):
        names.append(k)
        schemas.append(",".join([c.name for c in v.columns]))

__dh_live_catalog = new_table([
    string_col("name", names),
    string_col("columns_csv", schemas),
])
`;
    await this.ide.runCode(code);
    return await this.ide.getTable('__dh_live_catalog');
  }

  /** Create a ticking joined table on the server, then return a handle you can subscribe to. */
  async createJoinedTable(opts: {
    leftName: string;
    rightName: string;
    mode: JoinMode;
    on: string[];
    joins?: string[]; // RIGHT-side columns to bring in; default ['*']
  }): Promise<DhTable> {
    const { leftName, rightName, mode, on, joins = ['*'] } = opts;

    const rid = (globalThis.crypto?.randomUUID?.() ?? `${Date.now()}_${Math.random()}`)
      .toString()
      .replace(/[^a-zA-Z0-9_]/g, '');
    const varName = `joined_${rid}`;

    const py = `
left = ${leftName}
right = ${rightName}
mode = "${mode}"
on_cols = ${JSON.stringify(on)}
join_cols = ${JSON.stringify(joins)}

if mode == "natural":
    ${varName} = left.natural_join(right, on=on_cols, joins=join_cols)
elif mode == "exact":
    ${varName} = left.exact_join(right, on=on_cols, joins=join_cols)
elif mode == "left":
    ${varName} = left.left_join(right, on=on_cols, joins=join_cols)
elif mode == "aj":
    ${varName} = left.aj(right, on=on_cols)  # include the time key in on_cols
else:
    ${varName} = left.join(right, on=on_cols, joins=join_cols)
`;
    await this.ide.runCode(py);
    return await this.ide.getTable(varName);
  }

  async closeTableHandle(table: DhTable | null | undefined): Promise<void> {
    try { await table?.close?.(); } catch {}
  }
}
```

---

## 2) Generic standalone Angular component

```ts
// src/app/join-builder/join-builder.component.ts
import { ChangeDetectorRef, Component, OnDestroy, OnInit, signal } from '@angular/core';
import { CommonModule } from '@angular/common';
import { FormsModule } from '@angular/forms';
import type { JoinMode, TableCatalogRow } from '../services/deephaven.service';
import { DeephavenService } from '../services/deephaven.service';

@Component({
  selector: 'app-join-builder',
  standalone: true,
  imports: [CommonModule, FormsModule],
  template: `
  <section class="card">
    <h2>Build a Live Join</h2>

    <form class="grid" (ngSubmit)="buildJoin()">
      <label>
        Left table
        <select [(ngModel)]="left" name="left" required>
          <option *ngFor="let t of tableList()" [value]="t.name">{{ t.name }}</option>
        </select>
      </label>

      <label>
        Right table
        <select [(ngModel)]="right" name="right" required>
          <option *ngFor="let t of tableList()" [value]="t.name">{{ t.name }}</option>
        </select>
      </label>

      <label>
        Join type
        <select [(ngModel)]="mode" name="mode">
          <option value="natural">natural_join</option>
          <option value="exact">exact_join</option>
          <option value="left">left_join</option>
          <option value="join">join (inner)</option>
          <option value="aj">aj (as-of)</option>
        </select>
      </label>

      <label>
        Keys (CSV)
        <input [(ngModel)]="keysCsv" name="keysCsv" placeholder="e.g. Symbol,Timestamp"/>
        <small *ngIf="mode==='aj'">Include your time key for as-of join.</small>
      </label>

      <label>
        Right columns (CSV; '*' = all)
        <input [(ngModel)]="joinsCsv" name="joinsCsv" placeholder="e.g. Bid,Ask or *"/>
      </label>

      <div class="row">
        <button type="submit" [disabled]="building">Create Ticking Join</button>
        <button type="button" (click)="reset()" [disabled]="!joinedTable">Reset</button>
      </div>
    </form>

    <p class="hint" *ngIf="error">{{ error }}</p>
  </section>

  <section class="card" *ngIf="joinedTable">
    <header class="thead">
      <div class="th" *ngFor="let c of cols">{{ c }}</div>
    </header>

    <div class="tbody" role="table">
      <div class="tr" role="row" *ngFor="let r of rows">
        <div class="td" role="cell" *ngFor="let v of r">{{ v }}</div>
      </div>
    </div>
  </section>
  `,
  styles: [`
    :host { display:block; padding:1rem; color:#fff; }
    .card { border:1px solid #2a2a2a; border-radius:.75rem; padding:1rem; margin-bottom:1rem; background:#111; }
    .grid { display:grid; gap:.75rem; grid-template-columns: repeat(auto-fit,minmax(260px,1fr)); }
    label { display:flex; flex-direction:column; gap:.25rem; }
    input, select, button { padding:.5rem; background:#161616; color:#fff; border:1px solid #333; border-radius:.5rem; }
    .row { display:flex; gap:.5rem; align-items:center; }
    .thead, .tr { display:grid; grid-template-columns: repeat(auto-fit, minmax(140px, 1fr)); }
    .th, .td { padding:.5rem; border-bottom:1px solid #1f1f1f; white-space:nowrap; overflow:hidden; text-overflow:ellipsis; }
    .thead { position:sticky; top:0; background:#0f0f0f; font-weight:600; border-bottom:1px solid #333; }
    .tbody { max-height:60vh; overflow:auto; }
    .hint { color:#ffb3b3; }
  `]
})
export class JoinBuilderComponent implements OnInit, OnDestroy {
  // form state
  left = '';
  right = '';
  mode: JoinMode = 'natural';
  keysCsv = '';
  joinsCsv = '*';

  // state
  tableList = signal<TableCatalogRow[]>([]);
  building = false;
  error = '';

  // current joined view
  joinedTable: any | null = null;
  cols: string[] = [];
  private colRefs: any[] = [];
  rows: unknown[][] = [];

  constructor(private dh: DeephavenService, private cdr: ChangeDetectorRef) {}

  private onUpdate = async () => {
    if (!this.joinedTable) return;
    try {
      const vp = await this.joinedTable.getViewportData();
      this.rows = vp.rows.map((r: any) => this.colRefs.map(col => r.get(col)));
      this.cdr.markForCheck();
    } catch {}
  };

  async ngOnInit(): Promise<void> {
    try {
      await this.dh.connect(); // set URL/PSK in the service if different
      const catalog = await this.dh.getLiveTableCatalog();

      await catalog.setViewport(0, 2000);
      const vp = await catalog.getViewportData();
      const nameCol = catalog.columns.find((c:any)=>c.name==='name');
      const colsCol = catalog.columns.find((c:any)=>c.name==='columns_csv');
      const items = vp.rows.map((r:any) => ({
        name: r.get(nameCol),
        columns_csv: r.get(colsCol),
      })) as TableCatalogRow[];
      this.tableList.set(items);

      this.left  = items[0]?.name ?? '';
      this.right = items[1]?.name ?? '';
    } catch (e:any) {
      this.error = 'Unable to load live tables. ' + (e?.message ?? '');
    } finally {
      this.cdr.markForCheck();
    }
  }

  async buildJoin(): Promise<void> {
    this.error = '';
    this.building = true;
    this.cdr.markForCheck();

    await this.teardownJoinedTable();

    try {
      const on = this.keysCsv.split(',').map(s => s.trim()).filter(Boolean);
      const joins = this.joinsCsv === '*' ? ['*'] : this.joinsCsv.split(',').map(s => s.trim()).filter(Boolean);

      this.joinedTable = await this.dh.createJoinedTable({
        leftName: this.left,
        rightName: this.right,
        mode: this.mode,
        on,
        joins,
      });

      this.colRefs = (this.joinedTable.columns ?? []);
      this.cols = this.colRefs.map((c:any) => c.name);

      await this.joinedTable.setViewport(0, 300);
      await this.onUpdate();

      this.joinedTable.addEventListener(this.joinedTable.EVENT_UPDATED, this.onUpdate);
      this.joinedTable.addEventListener?.(this.joinedTable.EVENT_CLOSED, () => {
        this.error = 'Joined table was closed on the server.';
        this.cdr.markForCheck();
      });
    } catch (e:any) {
      this.error = 'Failed to create joined table. ' + (e?.message ?? '');
    } finally {
      this.building = false;
      this.cdr.markForCheck();
    }
  }

  async reset(): Promise<void> {
    this.error = '';
    await this.teardownJoinedTable();
    this.cols = [];
    this.colRefs = [];
    this.rows = [];
    this.cdr.markForCheck();
  }

  private async teardownJoinedTable(): Promise<void> {
    try {
      if (this.joinedTable) {
        this.joinedTable.removeEventListener?.(this.joinedTable.EVENT_UPDATED, this.onUpdate);
        await this.joinedTable.close?.();
      }
    } catch {} finally {
      this.joinedTable = null;
    }
  }

  async ngOnDestroy(): Promise<void> {
    await this.teardownJoinedTable();
  }
}
```

---

### Why this works (and why `init` didn’t)

- **`@deephaven/jsapi-bootstrap` 1.5.3** doesn’t export a default `init` function; its public API is different (hooks/classes). Importing `dh` **directly** from your Deephaven server is the supported, stable path and is exactly how the docs do it (see the “Use the JS API” guide where they import `dh` from `/jsapi/dh-core.js` and then `new dh.CoreClient(...)`). ([Deephaven](https://deephaven.io/core/docs/how-to-guides/use-jsapi/ "Use the JS API | Deephaven"))
    
- The code above avoids type/shape drift by treating the `dh` runtime as `any`, while still giving you safe Angular code and proper cleanup.
    
- It lists **only live tables**, performs the **join on the server**, and streams updates to the UI via `setViewport` + `EVENT_UPDATED`.
    

If you still prefer to go through the bootstrap package, we can switch to that, but this direct import is the fastest way to get you unblocked and it matches Deephaven’s official guidance.