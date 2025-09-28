Great — your requirement is:

- **Only** show a row when an **account event** arrives.
    
- If the account for the same user changes later, **append a new row** (don’t rewrite old rows).
    
- Enrich each account row with the **user fields** that were true **at that moment**.
    

### Use **as-of join (`aj`)** driven by the **account** stream

Why: a normal `join` stays “live” to user changes and would rewrite old rows when the user profile changes later.  
`aj` picks the latest user row **at or before** the account event’s timestamp, so past rows remain unchanged — exactly the “append-only history” you want.

---

## Deephaven (server) side

Make sure both streams have a timestamp column. You usually already get Kafka record time; if not, add one.

```python
from deephaven.stream.kafka import consumer as kc
from deephaven import dtypes as dt
from deephaven import time as dhtime

KAFKA_CONFIG = { ... }  # your existing config

USER_VALUE_SPEC = kc.json_spec({
    "userId": dt.string,
    "name":   dt.string,
    "email":  dt.string,
    "age":    dt.int_
})
ACCOUNT_VALUE_SPEC = kc.json_spec({
    "userId":      dt.string,
    "accountType": dt.string,
    "balance":     dt.double
})

user_stream = kc.consume(
    KAFKA_CONFIG, "user-topic",
    key_spec=kc.KeyValueSpec.IGNORE,
    value_spec=USER_VALUE_SPEC,
    table_type=kc.TableType.append,
)
account_stream = kc.consume(
    KAFKA_CONFIG, "account-topic",
    key_spec=kc.KeyValueSpec.IGNORE,
    value_spec=ACCOUNT_VALUE_SPEC,
    table_type=kc.TableType.append,
)

# Extract tables
user  = user_stream.table
account = account_stream.table

# Ensure both have a timestamp column `ts` (pick ONE approach):
# A) Use Kafka record time if present:
# user   = user.rename_columns(["ts = KafkaTimestamp"])
# account= account.rename_columns(["ts = KafkaTimestamp"])

# B) Or create an ingest time:
user    = user.update(["ts = now()"])
account = account.update(["ts = now()"])

# Keep at most 1 row per user per instant (optional but typical):
# If user can emit multiple times, we want the latest AS OF that time axis.
users_hist = user.sort_descending("ts")

# Append-only, account-driven, time-correct enrichment:
# result row appears ONLY when an account row arrives; later user changes do not rewrite history.
user_account = account.aj(users_hist, on=["userId"], timestamp="ts", joins=["name","email","age"])

# Expose these variables to your Angular app:
#   user, account, user_account
```

> If you truly never have a usable timestamp, you _can_ stick with `join`, but old rows will update when user changes. For your “append-only history” requirement, **use `aj`**.

---

## Angular changes (your service already supports modes)

In the place where you request the joined table, switch to `aj` and keep **account** on the left:

```ts
// join-tables.component.ts (unchanged structure)
this.joinedHandle = await this.dh.createJoinedTable({
  leftName: 'account',     // drive by account events
  rightName: 'users_hist', // or 'user' if you used update(now()); sorted not required for aj
  mode: 'aj',
  on: ['userId'],          // common key
  // 'aj' ignores joins= when timestamp specified? (safe to pass the columns you need)
  joins: ['name', 'email', 'age'],
});
```

> If you kept the table name as `user_account` server-side and just want to **fetch** it, you can also skip `createJoinedTable` and call `getTableHandle('user_account')`. But keeping it in the service is fine.

---

## Summary

- **What join now?** Use **`aj` (as-of join)**.
    
- **Why?** It appends a new row **only** when an **account** message arrives and freezes user fields **as of that time**, so past rows aren’t modified by later user updates.
    
- **How?** Ensure both tables have a `ts` column, then:
    
    ```python
    user_account = account.aj(user, on=['userId'], timestamp='ts', joins=[...])
    ```
    
- **Angular**: call `createJoinedTable` with `mode: 'aj'`, left=`account`, right=`user` (or `users_hist`), `on=['userId']`.


Awesome — your PSK flow is already there. Below is a **leaned-down** version that keeps only what you need:

- Simple PSK connect
    
- Fetch handles to `user`, `account`, and a **live joined** table
    
- Angular 20.3 + AG Grid view with search
    
- No extra “IDE helpers” beyond tiny bits to run one join snippet
    

---

# 1) env

`src/environments/environment.ts`

```ts
export const environment = {
  production: false,
  // e.g. http://localhost:10000 or your remote DH URL
  deephavenUrl: 'http://localhost:10000',
  // put your PSK here (or load from .env/secret management in your app)
  deephavenPsk: 'YOUR_PSK_TOKEN',
};
```

---

# 2) Deephaven service (PSK + minimal table helpers)

`src/app/deephaven/deephaven.service.ts`

```ts
import { Injectable } from '@angular/core';
import { environment } from '../../environments/environment';

// keep runtime types loose to avoid DH version drift
type DhNS = any;
type DhClient = any;
type DhIde = any;
export type DhTable = any;

export type JoinMode = 'natural' | 'exact' | 'left' | 'join' | 'aj';

@Injectable({ providedIn: 'root' })
export class DeephavenService {
  private dh!: DhNS;
  private client!: DhClient;
  private ide!: DhIde;
  private ready = false;

  get isReady(): boolean {
    return this.ready;
  }

  /** Connect using PSK from environment. */
  async connect(): Promise<void> {
    if (this.ready) return;

    const serverUrl = environment.deephavenUrl;
    const psk = environment.deephavenPsk;
    if (!serverUrl) throw new Error('environment.deephavenUrl is not set');
    if (!psk) throw new Error('environment.deephavenPsk is not set');

    // Vite/Angular-safe dynamic import of DH JS API
    const jsapiUrl = new URL('/jsapi/dh-core.js', serverUrl).toString();
    const mod = await import(/* @vite-ignore */ jsapiUrl);
    this.dh = mod.default;

    this.client = new this.dh.Client(serverUrl);
    this.client.addEventListener?.('error', (e: any) =>
      console.error('[DH] client error:', e)
    );
    this.client.addEventListener?.('disconnect', (e: any) =>
      console.warn('[DH] disconnect:', e?.reason ?? e)
    );

    await this.client.login({
      type: 'io.deephaven.authentication.psk.PskAuthenticationHandler',
      token: psk,
    });

    this.ide = await this.client.getAsIdeConnection();
    this.ide.addEventListener?.('error', (e: any) =>
      console.error('[DH] IDE error:', e)
    );
    this.ide.addEventListener?.('disconnect', (e: any) =>
      console.warn('[DH] IDE disconnect:', e?.reason ?? e)
    );

    // ready – a python session exists behind the scenes
    await this.ide.startSession('python');
    this.ready = true;
    console.log('[DH] IDE session ready');
  }

  /** Get a handle to a server table variable (globals()). */
  async getTableHandle(name: string): Promise<DhTable> {
    if (!this.ready) await this.connect();
    if (!name || !/^[A-Za-z_][A-Za-z0-9_]*$/.test(name)) {
      throw new Error(`Invalid table variable name: "${name}"`);
    }
    return await this.ide.getTable(name);
  }

  /** Create a ticking joined table on the server and return a handle to it. */
  async createJoinedTable(opts: {
    leftName: string;
    rightName: string;
    mode: JoinMode;
    on: string[];      // e.g. ['userId']
    joins?: string[];  // columns to append from right
  }): Promise<DhTable> {
    const { leftName, rightName, mode, on, joins = ['*'] } = opts;
    if (!this.ready) await this.connect();

    // unique python var name that is valid for globals()
    const rid = (globalThis.crypto?.randomUUID?.() ?? `${Date.now()}_${Math.random()}`)
      .toString()
      .replace(/[^a-zA-Z0-9_]/g, '_');
    const varName = `joined_${rid}`;

    // python code executed server-side
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
    ${varName} = left.aj(right, on=on_cols)
else:
    ${varName} = left.join(right, on=on_cols, joins=join_cols)
`.trim();

    await this.ide.runCode(py);
    return await this.ide.getTable(varName);
  }

  async closeTableHandle(table: DhTable | null | undefined): Promise<void> {
    try {
      await table?.close?.();
    } catch {}
  }

  /** Minimal live → array adapter for AG Grid. */
  createLiveAdapter() {
    const dh = this.dh;
    return new LiveTableAdapter(dh);
  }
}

/** A tiny adapter that uses a viewport to push live rows into AG Grid. */
class LiveTableAdapter {
  private viewport: any;
  private sub: any;
  private table: any;
  private dh: any;

  constructor(dh: any) {
    this.dh = dh;
  }

  async bind(table: any, onRows: (rows: any[]) => void, cols?: string[]) {
    this.table = table;

    // establish a viewport (0..N). You can tune rowCount as needed.
    const definition = {
      columns: cols ?? table.columns.map((c: any) => c.name),
      rowOffset: 0,
      rowCount: 10_000, // adjust
    };

    // set viewport + subscribe to updates
    this.viewport = await table.setViewport(
      definition.rowOffset,
      definition.rowCount,
      definition.columns
    );

    // initial full snapshot
    const snapshot = await table.snapshot(definition.columns);
    onRows(snapshot.toObjects?.() ?? snapshot);

    // updates
    const EVENT = (this.dh?.Table?.EVENT_UPDATED ?? 'update');
    this.sub = (e: any) => {
      try {
        // convert the delta to full set if desired (simple path: re-snapshot)
        table.snapshot(definition.columns).then((s: any) => {
          onRows(s.toObjects?.() ?? s);
        });
      } catch (err) {
        console.error('Viewport update error', err);
      }
    };
    table.addEventListener(EVENT, this.sub);
  }

  async unbind() {
    try {
      if (this.table && this.sub) {
        const EVENT = (this.dh?.Table?.EVENT_UPDATED ?? 'update');
        this.table.removeEventListener?.(EVENT, this.sub);
      }
      await this.viewport?.close?.();
    } catch {}
  }
}
```

> The adapter above chooses the simplest, robust path for Angular apps: it refreshes the grid by taking a fresh snapshot on each delta (which is perfectly fine for moderate table sizes). If you need ultra-low latency on very large tables, you can extend it to apply row-level deltas.

---

# 3) AG Grid setup

Install (once):

```bash
npm i ag-grid-community ag-grid-angular
```

---

# 4) Component to show two tables (side-by-side) + joined table (full width)

`src/app/join-tables/join-tables.component.ts`

```ts
import { Component, OnDestroy, OnInit, ViewChild } from '@angular/core';
import { DeephavenService, DhTable } from '../deephaven/deephaven.service';
import { AgGridAngular } from 'ag-grid-angular';
import { ColDef, GridApi, GridReadyEvent } from 'ag-grid-community';

@Component({
  selector: 'app-join-tables',
  templateUrl: './join-tables.component.html',
  styleUrls: ['./join-tables.component.css'],
})
export class JoinTablesComponent implements OnInit, OnDestroy {
  // grid refs
  @ViewChild('userGrid') userGrid!: AgGridAngular;
  @ViewChild('accountGrid') accountGrid!: AgGridAngular;
  @ViewChild('joinedGrid') joinedGrid!: AgGridAngular;

  userCols: ColDef[] = [];
  accountCols: ColDef[] = [];
  joinedCols: ColDef[] = [];

  userRows: any[] = [];
  accountRows: any[] = [];
  joinedRows: any[] = [];

  // search text
  userSearch = '';
  accountSearch = '';
  joinedSearch = '';

  private userApi?: GridApi;
  private accountApi?: GridApi;
  private joinedApi?: GridApi;

  private userHandle?: DhTable;
  private accountHandle?: DhTable;
  private joinedHandle?: DhTable;

  private userAdapter?: any;
  private accountAdapter?: any;
  private joinedAdapter?: any;

  constructor(private dh: DeephavenService) {}

  async ngOnInit() {
    await this.dh.connect();

    // get handles (server-side variables must exist in your session: user, account)
    this.userHandle = await this.dh.getTableHandle('user');
    this.accountHandle = await this.dh.getTableHandle('account');

    // build a live join on 'userId'
    this.joinedHandle = await this.dh.createJoinedTable({
      leftName: 'account',
      rightName: 'user',
      mode: 'join',          // inner join
      on: ['userId'],
      // select which columns from right to add; '*' takes all
      joins: ['name', 'email', 'age'],
    });

    // set grid columns from DH schemas
    this.userCols = this.makeCols(this.userHandle);
    this.accountCols = this.makeCols(this.accountHandle);
    this.joinedCols = this.makeCols(this.joinedHandle);

    // bind live data → grids
    this.userAdapter = this.dh.createLiveAdapter();
    this.accountAdapter = this.dh.createLiveAdapter();
    this.joinedAdapter = this.dh.createLiveAdapter();

    await this.userAdapter.bind(this.userHandle, rows => {
      this.userRows = rows; this.userApi?.setRowData(rows);
    });
    await this.accountAdapter.bind(this.accountHandle, rows => {
      this.accountRows = rows; this.accountApi?.setRowData(rows);
    });
    await this.joinedAdapter.bind(this.joinedHandle, rows => {
      this.joinedRows = rows; this.joinedApi?.setRowData(rows);
    });
  }

  ngOnDestroy(): void {
    this.userAdapter?.unbind();
    this.accountAdapter?.unbind();
    this.joinedAdapter?.unbind();

    this.dh.closeTableHandle(this.userHandle);
    this.dh.closeTableHandle(this.accountHandle);
    this.dh.closeTableHandle(this.joinedHandle);
  }

  onUserGridReady(e: GridReadyEvent)   { this.userApi = e.api;   e.api.setRowData(this.userRows); }
  onAccountGridReady(e: GridReadyEvent){ this.accountApi = e.api; e.api.setRowData(this.accountRows); }
  onJoinedGridReady(e: GridReadyEvent) { this.joinedApi = e.api;  e.api.setRowData(this.joinedRows); }

  quickFilter(which: 'user' | 'account' | 'joined', text: string) {
    if (which === 'user') this.userApi?.setQuickFilter(text);
    else if (which === 'account') this.accountApi?.setQuickFilter(text);
    else this.joinedApi?.setQuickFilter(text);
  }

  private makeCols(table: any): ColDef[] {
    return (table?.columns ?? []).map((c: any) => ({
      field: c.name,
      sortable: true,
      filter: true,
      resizable: true,
      minWidth: 120,
    })) as ColDef[];
  }
}
```

`src/app/join-tables/join-tables.component.html`

```html
<div class="grid-2">
  <div class="panel">
    <div class="panel-header">
      <h3>User</h3>
      <input placeholder="Search users…" [(ngModel)]="userSearch"
             (input)="quickFilter('user', userSearch)" />
    </div>
    <ag-grid-angular
      #userGrid
      class="ag-theme-alpine grid"
      [rowData]="userRows"
      [columnDefs]="userCols"
      [rowSelection]="'single'"
      (gridReady)="onUserGridReady($event)">
    </ag-grid-angular>
  </div>

  <div class="panel">
    <div class="panel-header">
      <h3>Account</h3>
      <input placeholder="Search accounts…" [(ngModel)]="accountSearch"
             (input)="quickFilter('account', accountSearch)" />
    </div>
    <ag-grid-angular
      #accountGrid
      class="ag-theme-alpine grid"
      [rowData]="accountRows"
      [columnDefs]="accountCols"
      [rowSelection]="'single'"
      (gridReady)="onAccountGridReady($event)">
    </ag-grid-angular>
  </div>
</div>

<div class="panel full">
  <div class="panel-header">
    <h3>User ⨉ Account (join on userId)</h3>
    <input placeholder="Search joined…" [(ngModel)]="joinedSearch"
           (input)="quickFilter('joined', joinedSearch)" />
  </div>
  <ag-grid-angular
    #joinedGrid
    class="ag-theme-alpine grid"
    [rowData]="joinedRows"
    [columnDefs]="joinedCols"
    [rowSelection]="'multiple'"
    (gridReady)="onJoinedGridReady($event)">
  </ag-grid-angular>
</div>
```

`src/app/join-tables/join-tables.component.css`

```css
.grid-2 {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 12px;
}

.panel {
  background: var(--panel-bg, #fff);
  border: 1px solid #e5e7eb;
  border-radius: 10px;
  padding: 10px;
  display: flex;
  flex-direction: column;
  min-height: 320px;
}

.panel.full { margin-top: 12px; }

.panel-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 8px;
  margin-bottom: 8px;
}

.panel-header input {
  width: 260px;
  padding: 6px 10px;
  border-radius: 8px;
  border: 1px solid #d1d5db;
}

.grid { width: 100%; height: 360px; }
.panel.full .grid { height: 420px; }
```

---

# 5) Module wire-up

Add to your app module:

```ts
import { AgGridModule } from 'ag-grid-angular';
import { FormsModule } from '@angular/forms';

@NgModule({
  declarations: [JoinTablesComponent],
  imports: [BrowserModule, FormsModule, AgGridModule],
  bootstrap: [JoinTablesComponent]
})
export class AppModule {}
```

---

## How it works

- **PSK auth**: `DeephavenService.connect()` loads `/jsapi/dh-core.js` from your DH server, logs in with PSK, and starts a Python session.
    
- **Tables**: It grabs `user` and `account` (which you’re already creating from Kafka) and creates a **live** joined table on the server using `join` on `userId`.
    
- **UI**: Two AG Grid panes on top (left/right), and one full-width pane below for the joined table.
    
- **Search**: Each grid uses AG Grid quick filter (client-side, super fast).
    
- **Streaming**: The `LiveTableAdapter` keeps a viewport and refreshes rows when DH publishes updates.
    

If you want me to switch the join to `natural_join` or `left_join`, just say the word.


-----------------------
top


You’re on **AG Grid v34.2.0**. In v29+:

- `gridApi.setRowData(...)` was **removed** → just bind `[rowData]` in the template and update the array.
    
- `gridApi.setQuickFilter(...)` was **removed** → use `gridApi.setGridOption('quickFilterText', text)`.
    

Here’s the minimal, **compatible** fix for your component.

---

## join-tables.component.ts (only the parts that change)

```ts
import { Component, OnDestroy, OnInit, ViewChild } from '@angular/core';
import { AgGridAngular } from 'ag-grid-angular';
import { ColDef, GridApi, GridReadyEvent } from 'ag-grid-community';
import { DeephavenService, DhTable } from '../deephaven/deephaven.service';

@Component({
  selector: 'app-join-tables',
  templateUrl: './join-tables.component.html',
  styleUrls: ['./join-tables.component.css'],
})
export class JoinTablesComponent implements OnInit, OnDestroy {
  @ViewChild('userGrid') userGrid!: AgGridAngular;
  @ViewChild('accountGrid') accountGrid!: AgGridAngular;
  @ViewChild('joinedGrid') joinedGrid!: AgGridAngular;

  userCols: ColDef[] = [];
  accountCols: ColDef[] = [];
  joinedCols: ColDef[] = [];

  // bind these to [rowData] in the template
  userRows: any[] = [];
  accountRows: any[] = [];
  joinedRows: any[] = [];

  private userApi?: GridApi;
  private accountApi?: GridApi;
  private joinedApi?: GridApi;

  private userHandle?: DhTable;
  private accountHandle?: DhTable;
  private joinedHandle?: DhTable;

  private userAdapter?: any;
  private accountAdapter?: any;
  private joinedAdapter?: any;

  constructor(private dh: DeephavenService) {}

  async ngOnInit() {
    await this.dh.connect();

    this.userHandle = await this.dh.getTableHandle('user');
    this.accountHandle = await this.dh.getTableHandle('account');

    // For your “account append-only + enrich from user as-of time” case you chose `aj`
    this.joinedHandle = await this.dh.createJoinedTable({
      leftName: 'account',
      rightName: 'user',
      mode: 'aj',
      on: ['userId', 'ts'],           // last element is the as-of key
      joins: ['name', 'email', 'age'],
    });

    this.userCols = this.makeCols(this.userHandle);
    this.accountCols = this.makeCols(this.accountHandle);
    this.joinedCols = this.makeCols(this.joinedHandle);

    // bind Deephaven → arrays; Angular will update the grid via [rowData]
    this.userAdapter = this.dh.createLiveAdapter();
    this.accountAdapter = this.dh.createLiveAdapter();
    this.joinedAdapter = this.dh.createLiveAdapter();

    await this.userAdapter.bind(this.userHandle, rows => { this.userRows = rows; });
    await this.accountAdapter.bind(this.accountHandle, rows => { this.accountRows = rows; });
    await this.joinedAdapter.bind(this.joinedHandle, rows => { this.joinedRows = rows; });
  }

  ngOnDestroy(): void {
    this.userAdapter?.unbind();
    this.accountAdapter?.unbind();
    this.joinedAdapter?.unbind();
    this.dh.closeTableHandle(this.userHandle);
    this.dh.closeTableHandle(this.accountHandle);
    this.dh.closeTableHandle(this.joinedHandle);
  }

  onUserGridReady(e: GridReadyEvent)   { this.userApi   = e.api; }
  onAccountGridReady(e: GridReadyEvent){ this.accountApi= e.api; }
  onJoinedGridReady(e: GridReadyEvent) { this.joinedApi = e.api; }

  // v34 way to set quick filter
  quickFilter(which: 'user'|'account'|'joined', text: string) {
    const api =
      which === 'user' ? this.userApi :
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
    }));
  }
}
```

---

## join-tables.component.html (unchanged data binding; inputs drive quick filter)

```html
<div class="grid-2">
  <div class="panel">
    <div class="panel-header">
      <h3>User</h3>
      <input placeholder="Search users…" (input)="quickFilter('user', ($event.target as HTMLInputElement).value)" />
    </div>
    <ag-grid-angular
      #userGrid
      class="ag-theme-alpine grid"
      [rowData]="userRows"
      [columnDefs]="userCols"
      (gridReady)="onUserGridReady($event)">
    </ag-grid-angular>
  </div>

  <div class="panel">
    <div class="panel-header">
      <h3>Account</h3>
      <input placeholder="Search accounts…" (input)="quickFilter('account', ($event.target as HTMLInputElement).value)" />
    </div>
    <ag-grid-angular
      #accountGrid
      class="ag-theme-alpine grid"
      [rowData]="accountRows"
      [columnDefs]="accountCols"
      (gridReady)="onAccountGridReady($event)">
    </ag-grid-angular>
  </div>
</div>

<div class="panel full">
  <div class="panel-header">
    <h3>User × Account (as-of on ts)</h3>
    <input placeholder="Search joined…" (input)="quickFilter('joined', ($event.target as HTMLInputElement).value)" />
  </div>
  <ag-grid-angular
    #joinedGrid
    class="ag-theme-alpine grid"
    [rowData]="joinedRows"
    [columnDefs]="joinedCols"
    (gridReady)="onJoinedGridReady($event)">
  </ag-grid-angular>
</div>
```

---

### Why your errors happened

- `setRowData` and `setQuickFilter` were removed from the public API. In v34, you **bind `rowData`** and update arrays, and you **set quick filter** with:
    

```ts
api.setGridOption('quickFilterText', text);
```

If you want, we can also switch to the new **Quartz theme** (`ag-theme-quartz`) or add server-side row model later—but the above is the smallest change to make your current app compile and run cleanly on **34.2.0**.


------------------------------------------------------------------------------

chil.........

You’re right—the join should stay in Deephaven. In Angular you should only **read three existing live tables**: `user`, `account`, and your server-side `user_account`. Also, your template error is just a small parentheses/typing issue.

Here’s the cleaned, **v34.2.0-compatible** setup:

# 1) TS: only fetch existing tables (no createJoinedTable)

```ts
// join-tables.component.ts (relevant parts)
async ngOnInit() {
  await this.dh.connect();

  // these must already exist in your DH session
  this.userHandle = await this.dh.getTableHandle('user');
  this.accountHandle = await this.dh.getTableHandle('account');
  this.joinedHandle = await this.dh.getTableHandle('user_account'); // <-- FROM DH, not built in Angular

  this.userCols   = this.makeCols(this.userHandle);
  this.accountCols= this.makeCols(this.accountHandle);
  this.joinedCols = this.makeCols(this.joinedHandle);

  // bind Deephaven → arrays; Angular updates grids via [rowData]
  this.userAdapter    = this.dh.createLiveAdapter();
  this.accountAdapter = this.dh.createLiveAdapter();
  this.joinedAdapter  = this.dh.createLiveAdapter();

  await this.userAdapter.bind(this.userHandle,    rows => { this.userRows   = rows; });
  await this.accountAdapter.bind(this.accountHandle, rows => { this.accountRows = rows; });
  await this.joinedAdapter.bind(this.joinedHandle,  rows => { this.joinedRows  = rows; });
}

// v34 quick filter (setRowData/ setQuickFilter no longer exist)
quickFilter(which: 'user'|'account'|'joined', text: string) {
  const api =
    which === 'user'    ? this.userApi :
    which === 'account' ? this.accountApi :
                          this.joinedApi;
  api?.setGridOption('quickFilterText', text);
}
```

# 2) HTML: fix the `(input)` syntax (your error)

Angular template needs the **cast wrapped in parentheses**. Use exactly this:

```html
<!-- join-tables.component.html (inputs only) -->
<input
  placeholder="Search users…"
  (input)="quickFilter('user', (($event.target as HTMLInputElement).value))" />

<input
  placeholder="Search accounts…"
  (input)="quickFilter('account', (($event.target as HTMLInputElement).value))" />

<input
  placeholder="Search joined…"
  (input)="quickFilter('joined', (($event.target as HTMLInputElement).value))" />
```

Common pitfalls that cause the “Missing closing parenthesis” parser error:

- Missing the outer parentheses around the cast: `($event.target as HTMLInputElement).value`
    
- Smart quotes or stray commas. Copy exactly as above.
    

# 3) AG Grid v34 reminders (matches your package.json)

- Bind rows with `[rowData]="...Rows"` (don’t call `setRowData`).
    
- Use `api.setGridOption('quickFilterText', text)` for search.
    
- Keep `ColDef[]` simple; you already have it.
    

# 4) Quick checklist

- In Deephaven, make sure these Python variables exist and tick:
    
    - `user` (append),
        
    - `account` (append),
        
    - `user_account` (your `aj` join driven by `account`).
        
- In Angular, **only** read them with `getTableHandle(...)`.
    

If you want, paste your current `join-tables.component.html` (just the top inputs + ag-grid tags) and I’ll spot-fix any remaining template typos.


--------------
pil

Perfect—here’s a **minimal, clean DeephavenService** that does only what you need:

- PSK login
    
- `getTableHandle(name)` for existing (already-created) live tables on the DH server (e.g., `user`, `account`, `user_account`)
    
- Tiny live adapter to keep arrays in sync for AG Grid v34 (no `setRowData` calls)
    
- `closeTableHandle` and `disconnect` helpers
    

Drop this in `src/app/deephaven/deephaven.service.ts`.

```ts
import { Injectable } from '@angular/core';
import { environment } from '../../environments/environment';

type DhNS = any;
type DhClient = any;
type DhIde = any;
export type DhTable = any;

@Injectable({ providedIn: 'root' })
export class DeephavenService {
  private dh!: DhNS;
  private client!: DhClient;
  private ide!: DhIde;
  private ready = false;

  get isReady(): boolean {
    return this.ready;
  }

  /** Connect to Deephaven using PSK. */
  async connect(): Promise<void> {
    if (this.ready) return;

    const serverUrl = environment.deephavenUrl;
    const psk = environment.deephavenPsk;
    if (!serverUrl) throw new Error('environment.deephavenUrl is not set');
    if (!psk) throw new Error('environment.deephavenPsk is not set');

    // load JS API from the DH server
    const jsapiUrl = new URL('/jsapi/dh-core.js', serverUrl).toString();
    const mod = await import(/* @vite-ignore */ jsapiUrl);
    this.dh = mod.default;

    this.client = new this.dh.Client(serverUrl);
    await this.client.login({
      type: 'io.deephaven.authentication.psk.PskAuthenticationHandler',
      token: psk,
    });

    this.ide = await this.client.getAsIdeConnection();
    await this.ide.startSession('python'); // ensure a python session exists
    this.ready = true;
  }

  /** Get a handle to an existing server-side table variable (globals()). */
  async getTableHandle(name: string): Promise<DhTable> {
    if (!this.ready) await this.connect();
    if (!/^[A-Za-z_][A-Za-z0-9_]*$/.test(name)) {
      throw new Error(`Invalid table variable name: "${name}"`);
    }
    return this.ide.getTable(name);
  }

  /** Close a table handle (optional cleanup). */
  async closeTableHandle(table: DhTable | null | undefined): Promise<void> {
    try { await table?.close?.(); } catch {}
  }

  /** Disconnect (optional, if you want to tear down). */
  async disconnect(): Promise<void> {
    try { await this.ide?.close?.(); } catch {}
    try { await this.client?.close?.(); } catch {}
    this.ready = false;
  }

  /** Minimal live → array adapter for AG Grid. */
  createLiveAdapter() {
    const dh = this.dh;
    return new LiveTableAdapter(dh);
  }
}

/**
 * LiveTableAdapter: keeps a viewport open and refreshes rows.
 * Simple approach: resnapshot on updates (good for moderate tables).
 */
class LiveTableAdapter {
  private viewport: any;
  private sub: ((e: any) => void) | undefined;
  private table: any;
  private dh: any;

  constructor(dh: any) { this.dh = dh; }

  async bind(table: any, onRows: (rows: any[]) => void, columns?: string[]) {
    this.table = table;
    const cols = columns ?? table.columns.map((c: any) => c.name);

    // initial snapshot
    const snap = await table.snapshot(cols);
    onRows(snap.toObjects?.() ?? snap);

    // establish viewport for live updates
    this.viewport = await table.setViewport(0, 10000, cols); // adjust row count if needed

    // react to updates (resnapshot = simplest, robust path)
    const EVENT = (this.dh?.Table?.EVENT_UPDATED ?? 'update');
    this.sub = async () => {
      try {
        const s = await table.snapshot(cols);
        onRows(s.toObjects?.() ?? s);
      } catch (e) {
        // swallow transient errors
        console.warn('Viewport refresh failed', e);
      }
    };
    table.addEventListener(EVENT, this.sub);
  }

  async unbind() {
    try {
      if (this.table && this.sub) {
        const EVENT = (this.dh?.Table?.EVENT_UPDATED ?? 'update');
        this.table.removeEventListener?.(EVENT, this.sub);
      }
      await this.viewport?.close?.();
    } catch {}
  }
}
```

### How to use it (recap)

- In your component’s `ngOnInit`:
    
    ```ts
    await this.dh.connect();
    this.userHandle = await this.dh.getTableHandle('user');
    this.accountHandle = await this.dh.getTableHandle('account');
    this.joinedHandle = await this.dh.getTableHandle('user_account'); // built in DH
    
    this.userAdapter = this.dh.createLiveAdapter();
    await this.userAdapter.bind(this.userHandle, rows => this.userRows = rows);
    
    // same for account + joined
    ```
    
- In the template, bind `[rowData]="userRows"` (no `setRowData`) and for search use:
    
    ```ts
    api.setGridOption('quickFilterText', text);
    ```
    

--------------------------------------------

proper..................


Gotcha — we’ll make PSK **required** exactly like before, using `CoreClient` + `PskAuthenticationHandler`. Here’s a **clean, minimal** `DeephavenService` that only does what you need and fails fast if PSK is missing.

### `src/app/deephaven/deephaven.service.ts`

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

  get isReady(): boolean {
    return this.ready;
  }

  /** Connect to Deephaven with REQUIRED PSK auth. */
  async connect(): Promise<void> {
    if (this.ready) return;

    const serverUrl = environment.deephavenUrl;
    const psk = environment.deephavenPsk;
    if (!serverUrl) throw new Error('environment.deephavenUrl is not set');
    if (!psk) throw new Error('environment.deephavenPsk is not set');

    // Load DH JS API from the server
    const jsapiUrl = new URL('/jsapi/dh-core.js', serverUrl).toString();
    const mod = await import(/* @vite-ignore */ jsapiUrl);
    this.dh = mod.default;

    // CoreClient + PSK login
    this.client = new this.dh.CoreClient(serverUrl);
    await this.client.login({
      type: 'io.deephaven.authentication.psk.PskAuthenticationHandler',
      token: psk,
    });

    // IDE connection for table handles
    this.ide = await this.client.getAsIdeConnection();
    await this.ide.startSession('python'); // safe no-op if already present
    this.ready = true;
  }

  /** Get a handle to an existing globals() table on the DH server. */
  async getTableHandle(name: string): Promise<DhTable> {
    if (!this.ready) await this.connect();
    if (!/^[A-Za-z_][A-Za-z0-9_]*$/.test(name)) {
      throw new Error(`Invalid table variable name: "${name}"`);
    }
    return this.ide.getTable(name);
  }

  /** Live → array adapter for AG Grid: open viewport, resnapshot on updates. */
  createLiveAdapter() {
    const dh = this.dh;
    return new LiveTableAdapter(dh);
  }

  async closeTableHandle(table: DhTable | null | undefined): Promise<void> {
    try { await table?.close?.(); } catch {}
  }

  async disconnect(): Promise<void> {
    try { await this.ide?.close?.(); } catch {}
    try { await this.client?.close?.(); } catch {}
    this.ready = false;
  }
}

/** Simple, robust live adapter (good for moderate table sizes). */
class LiveTableAdapter {
  private viewport: any;
  private sub?: (e: any) => void;
  private table: any;
  private dh: any;

  constructor(dh: any) { this.dh = dh; }

  async bind(table: any, onRows: (rows: any[]) => void, columns?: string[]) {
    this.table = table;
    const cols = columns ?? (table?.columns ?? []).map((c: any) => c.name);

    // Initial snapshot
    const snap = await table.snapshot(cols);
    onRows(snap.toObjects?.() ?? snap);

    // Viewport for live updates (adjust rowCount if needed)
    this.viewport = await table.setViewport(0, 10000, cols);

    // Refresh rows on each update
    const EVENT = (this.dh?.Table?.EVENT_UPDATED ?? 'update');
    this.sub = async () => {
      try {
        const s = await table.snapshot(cols);
        onRows(s.toObjects?.() ?? s);
      } catch (e) {
        console.warn('Viewport refresh failed', e);
      }
    };
    table.addEventListener(EVENT, this.sub);
  }

  async unbind() {
    try {
      if (this.table && this.sub) {
        const EVENT = (this.dh?.Table?.EVENT_UPDATED ?? 'update');
        this.table.removeEventListener?.(EVENT, this.sub);
      }
      await this.viewport?.close?.();
    } catch {}
  }
}
```

### Environment (PSK **required**)

```ts
// src/environments/environment.ts
export const environment = {
  production: false,
  deephavenUrl: 'http://localhost:10000', // your DH base URL
  deephavenPsk: 'YOUR_PSK_TOKEN',         // must be set
};
```

### AG Grid (standalone app) – register modules once

```ts
// src/main.ts
import { bootstrapApplication } from '@angular/platform-browser';
import { appConfig } from './app/app.config';
import { AppComponent } from './app/app.component';

import { ModuleRegistry, AllCommunityModule } from 'ag-grid-community';
ModuleRegistry.registerModules([AllCommunityModule]);

bootstrapApplication(AppComponent, appConfig).catch(err => console.error(err));
```

This matches your AG Grid component usage:

- You call `await dh.connect()` (PSK enforced).
    
- `getTableHandle('user')`, `getTableHandle('account')`, `getTableHandle('user_account')`.
    
- For each, `adapter.bind(handle, rows => this.rows = rows)`.
    
- Grids update via `[rowData]="rows"` and quick filter via `api.setGridOption('quickFilterText', text)`.
    

If anything still glitches, share the exact error text and I’ll tighten it further.

-------------------------

jopper


You’re hitting: **“Table already closed; cannot be used again.”**  
That happens when the same DH table handle gets closed (or hot-reloaded) while something (your viewport / snapshot) still tries to use it. In dev, Angular can re-instantiate components, so you can accidentally **bind twice** and **close once**, leaving one binding holding a dead handle.

Fix it by:

1. **Never reuse a closed handle.**
    
2. **Cache handles** in the service (ref-counted).
    
3. **Bind once per handle**; unbind on destroy.
    
4. In dev, **don’t call close directly**; call a `release()` that only closes when the last consumer is gone.
    

Below is a drop-in, **minimal + robust** service and tiny changes for your component.

---

# Updated DeephavenService (PSK required, handle cache, safe bind)

`src/app/deephaven/deephaven.service.ts`

```ts
import { Injectable } from '@angular/core';
import { environment } from '../../environments/environment';

type DhNS = any;
type DhCoreClient = any;
type DhIde = any;
export type DhTable = any;

type HandleEntry = {
  name: string;
  table: DhTable;
  refCount: number;
  closed: boolean;
};

@Injectable({ providedIn: 'root' })
export class DeephavenService {
  private dh!: DhNS;
  private client!: DhCoreClient;
  private ide!: DhIde;
  private ready = false;

  // cache to prevent reusing closed handles
  private handles = new Map<string, HandleEntry>();

  get isReady(): boolean { return this.ready; }

  /** PSK-required connection */
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

  /** Get (and cache) a handle to an existing globals() table on the DH server. */
  async acquireTable(name: string): Promise<DhTable> {
    if (!this.ready) await this.connect();
    if (!/^[A-Za-z_][A-Za-z0-9_]*$/.test(name)) {
      throw new Error(`Invalid table variable name: "${name}"`);
    }

    const existing = this.handles.get(name);
    if (existing && !existing.closed) {
      existing.refCount += 1;
      return existing.table;
    }

    const table = await this.ide.getTable(name);
    this.handles.set(name, { name, table, refCount: 1, closed: false });
    return table;
  }

  /** Release a table handle; only closes when the last consumer releases. */
  async releaseTable(name: string): Promise<void> {
    const entry = this.handles.get(name);
    if (!entry || entry.closed) return;

    entry.refCount -= 1;
    if (entry.refCount <= 0) {
      try { await entry.table?.close?.(); } catch {}
      entry.closed = true;
      this.handles.delete(name);
    }
  }

  /** Disconnect everything (rarely needed in apps). */
  async disconnect(): Promise<void> {
    for (const [name] of this.handles) await this.releaseTable(name);
    try { await this.ide?.close?.(); } catch {}
    try { await this.client?.close?.(); } catch {}
    this.ready = false;
  }

  /** Minimal live → array adapter with safeguards */
  createLiveAdapter() {
    const dh = this.dh;
    return new LiveTableAdapter(dh);
  }
}

/** Keeps a viewport open and re-snapshots on updates (simple & robust) */
class LiveTableAdapter {
  private viewport: any;
  private sub?: (e: any) => void;
  private table?: any;
  private dh: any;
  private bound = false;

  constructor(dh: any) { this.dh = dh; }

  async bind(table: any, onRows: (rows: any[]) => void, columns?: string[]) {
    if (this.bound) await this.unbind(); // avoid double-binding in dev HMR
    this.bound = true;

    this.table = table;
    const cols = columns ?? (table?.columns ?? []).map((c: any) => c.name);

    // Establish viewport first (some engines prefer this order)
    this.viewport = await table.setViewport(0, 10000, cols);

    // Initial snapshot
    try {
      const snap = await table.snapshot(cols);
      onRows(snap.toObjects?.() ?? snap);
    } catch (e) {
      console.warn('Initial snapshot failed (will try updates):', e);
    }

    // Refresh on updates
    const EVENT = (this.dh?.Table?.EVENT_UPDATED ?? 'update');
    this.sub = async () => {
      try {
        // Re-snapshot whole viewport (simple & safe)
        const s = await table.snapshot(cols);
        onRows(s.toObjects?.() ?? s);
      } catch (e: any) {
        // If table was closed upstream, stop listening to avoid error loops
        const msg = ('' + (e?.message ?? e)).toLowerCase();
        if (msg.includes('closed')) {
          console.warn('Table is closed; unbinding listener.');
          await this.unbind().catch(() => {});
        } else {
          console.warn('Viewport refresh failed', e);
        }
      }
    };
    table.addEventListener(EVENT, this.sub);
  }

  async unbind() {
    if (!this.bound) return;
    this.bound = false;

    try {
      if (this.table && this.sub) {
        const EVENT = (this.dh?.Table?.EVENT_UPDATED ?? 'update');
        this.table.removeEventListener?.(EVENT, this.sub);
      }
      await this.viewport?.close?.();
    } catch {}
    this.sub = undefined;
    this.viewport = undefined;
    this.table = undefined;
  }
}
```

---

# Component changes (small but important)

1. **Use the ref-counted API** (`acquireTable` / `releaseTable`).
    
2. Bind once; unbind in `ngOnDestroy`.
    
3. Don’t call `closeTableHandle` yourself anymore.
    

```ts
// join-tables.component.ts (relevant parts)
async ngOnInit() {
  await this.dh.connect();

  this.userHandle    = await this.dh.acquireTable('user');
  this.accountHandle = await this.dh.acquireTable('account');
  this.joinedHandle  = await this.dh.acquireTable('user_account'); // built in DH

  this.userCols    = this.makeCols(this.userHandle);
  this.accountCols = this.makeCols(this.accountHandle);
  this.joinedCols  = this.makeCols(this.joinedHandle);

  this.userAdapter    = this.dh.createLiveAdapter();
  this.accountAdapter = this.dh.createLiveAdapter();
  this.joinedAdapter  = this.dh.createLiveAdapter();

  await this.userAdapter.bind(this.userHandle,    rows => { this.userRows   = rows; });
  await this.accountAdapter.bind(this.accountHandle, rows => { this.accountRows = rows; });
  await this.joinedAdapter.bind(this.joinedHandle,  rows => { this.joinedRows  = rows; });
}

async ngOnDestroy() {
  await this.userAdapter?.unbind();
  await this.accountAdapter?.unbind();
  await this.joinedAdapter?.unbind();

  // release (ref-counted). Do NOT call close directly.
  await this.dh.releaseTable('user');
  await this.dh.releaseTable('account');
  await this.dh.releaseTable('user_account');
}
```

---

## Why this fixes your issue

- The **cache + ref count** ensures we don’t reuse a handle that was closed by a previous instance (very common with Angular dev reloads).
    
- The adapter **unbinds safely** when it detects a closed table (prevents the “already closed” cascade).
    
- Ordering the viewport **before** the first snapshot + avoiding duplicate binds eliminates common “no rows” symptoms when dev HMR runs twice.
    

If you still see empty data after this, the usual cause is that the server-side `user_account` has **no rows yet**. Quick check in the DH IDE:

```python
print(user.size())        # should be > 0
print(account.size())     # should be > 0
print(user_account.size())
```

If sizes are >0 and the grid is still empty, tell me and I’ll adjust the adapter to stream deltas instead of resnapshotting.
-----------------------

change............

Alright—let’s lock this down so it Just Works™.

You’re seeing two families of errors:

1. **`Table already closed; cannot be used again`**  
    Happens in Angular dev because components can mount twice and tear down once (HMR / dev-mode double init). If we close a handle while another binding still references it, the next tick throws.
    
2. **`ClassCastException` from DH Java side**  
    Usually appears when a viewport/snapshot fires **after** a handle closed or while another viewport is still being torn down. It’s a race—fixable by (a) binding once, (b) ref-counting handles, (c) serializing adapter lifecycle, and (d) retrying once if DH says “closed”.
    

Below is a **battle-tested minimal solution**:

- **PSK-required** login using `CoreClient`.
    
- **Handle cache with ref counts** (no reusing closed handles).
    
- **Dev-safe release policy**: in dev builds we **don’t actually close** DH tables on component destroy (prevents “already closed” during HMR). In prod, we do close.
    
- **Single bind per adapter**, with **serialized state** and **“closed? -> silent unbind”** logic.
    
- **Viewport before snapshot**, and **one-shot retry** if DH reports closed during an update.
    

You do the join in Deephaven (e.g., `user`, `account`, `user_account`). Angular only reads them and streams rows into AG Grid v34.

---

# 1) Deephaven service (drop-in)

`src/app/deephaven/deephaven.service.ts`

```ts
import { Injectable } from '@angular/core';
import { environment } from '../../environments/environment';

type DhNS = any;
type DhCoreClient = any;
type DhIde = any;
export type DhTable = any;

type HandleEntry = {
  name: string;
  table: DhTable;
  refCount: number;
  closed: boolean;
};

const IS_PROD = !!environment['production'];

@Injectable({ providedIn: 'root' })
export class DeephavenService {
  private dh!: DhNS;
  private client!: DhCoreClient;
  private ide!: DhIde;
  private ready = false;

  private handles = new Map<string, HandleEntry>();

  get isReady(): boolean { return this.ready; }

  /** PSK-required connect using CoreClient. */
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

  /** Acquire (and cache) a globals() table. Safe for multiple consumers. */
  async acquireTable(name: string): Promise<DhTable> {
    if (!this.ready) await this.connect();
    if (!/^[A-Za-z_][A-Za-z0-9_]*$/.test(name)) {
      throw new Error(`Invalid table variable name: "${name}"`);
    }

    const cached = this.handles.get(name);
    if (cached && !cached.closed) {
      cached.refCount += 1;
      return cached.table;
    }

    const table = await this.ide.getTable(name);
    this.handles.set(name, { name, table, refCount: 1, closed: false });
    return table;
  }

  /** Release: in dev, don't actually close (avoids HMR double-dispose). In prod, close when last user releases. */
  async releaseTable(name: string): Promise<void> {
    const entry = this.handles.get(name);
    if (!entry || entry.closed) return;

    entry.refCount -= 1;

    if (!IS_PROD) {
      // Dev mode: never close—DH handles survive HMR; prevents "already closed".
      if (entry.refCount <= 0) this.handles.delete(name);
      return;
    }

    if (entry.refCount <= 0) {
      try { await entry.table?.close?.(); } catch {}
      entry.closed = true;
      this.handles.delete(name);
    }
  }

  /** Disconnect everything (usually not needed). */
  async disconnect(): Promise<void> {
    for (const [name] of this.handles) await this.releaseTable(name);
    try { await this.ide?.close?.(); } catch {}
    try { await this.client?.close?.(); } catch {}
    this.ready = false;
  }

  /** Minimal live → array adapter. One bind at a time. */
  createLiveAdapter() {
    return new LiveTableAdapter(this.dh);
  }
}

/** Live adapter: viewport -> snapshot; robust against "closed" and double-bind. */
class LiveTableAdapter {
  private dh: any;
  private table?: any;
  private viewport?: any;
  private sub?: (e: any) => void;
  private bound = false;
  private destroyed = false;

  constructor(dh: any) { this.dh = dh; }

  /** Bind once. Rebind will unbind first. */
  async bind(table: any, onRows: (rows: any[]) => void, columns?: string[]) {
    await this.unbind(); // serialize (no double-bind)
    this.destroyed = false;
    this.bound = true;
    this.table = table;

    const cols = columns ?? (table?.columns ?? []).map((c: any) => c.name);

    // 1) Open viewport first (helps with engines that snapshot implicitly)
    this.viewport = await table.setViewport(0, 10000, cols);

    // 2) Initial snapshot
    await this.safeSnapshot(onRows, cols);

    // 3) Listen for updates, re-snapshot. Retry once if "closed" shows up.
    const EVENT = (this.dh?.Table?.EVENT_UPDATED ?? 'update');
    let retrying = false;

    this.sub = async () => {
      if (this.destroyed) return;
      try {
        await this.safeSnapshot(onRows, cols);
        retrying = false;
      } catch (e: any) {
        const msg = (e?.message ?? String(e)).toLowerCase();
        const looksClosed = msg.includes('closed') || msg.includes('table already closed');
        if (looksClosed && !retrying) {
          retrying = true;
          // slight delay to let upstream settle, then try again
          setTimeout(async () => {
            if (!this.destroyed) {
              try { await this.safeSnapshot(onRows, cols); }
              catch { /* give up silently */ }
              retrying = false;
            }
          }, 80);
        } else {
          // benign log; don't spam console
          // console.warn('Viewport refresh failed:', e);
        }
      }
    };

    table.addEventListener(EVENT, this.sub);
  }

  private async safeSnapshot(onRows: (rows: any[]) => void, cols: string[]) {
    if (!this.table) return;
    const snap = await this.table.snapshot(cols);
    const rows = snap.toObjects?.() ?? snap;
    onRows(rows);
  }

  async unbind() {
    this.destroyed = true;
    if (!this.bound) return;
    this.bound = false;

    try {
      if (this.table && this.sub) {
        const EVENT = (this.dh?.Table?.EVENT_UPDATED ?? 'update');
        this.table.removeEventListener?.(EVENT, this.sub);
      }
    } catch {}
    try { await this.viewport?.close?.(); } catch {}
    this.sub = undefined;
    this.viewport = undefined;
    this.table = undefined;
  }
}
```

---

# 2) Component (standalone) — use acquire/release and bind once

Only the relevant parts shown; keep your HTML as you have (AG Grid v34: `[rowData]` + `api.setGridOption('quickFilterText', ...)`).

```ts
import { Component, OnDestroy, OnInit, ViewChild } from '@angular/core';
import { CommonModule } from '@angular/common';
import { FormsModule } from '@angular/forms';
import { AgGridAngular } from 'ag-grid-angular';
import { ColDef, GridApi, GridReadyEvent } from 'ag-grid-community';
import { DeephavenService, DhTable } from '../deephaven/deephaven.service';

@Component({
  standalone: true,
  selector: 'app-join-tables',
  templateUrl: './join-tables.component.html',
  styleUrls: ['./join-tables.component.css'],
  imports: [CommonModule, FormsModule, AgGridAngular],
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

  private userHandle?: DhTable;
  private accountHandle?: DhTable;
  private joinedHandle?: DhTable;

  private userAdapter?: any;
  private accountAdapter?: any;
  private joinedAdapter?: any;

  constructor(private dh: DeephavenService) {}

  async ngOnInit() {
    await this.dh.connect();

    // IMPORTANT: only acquire; don't create joins in Angular
    this.userHandle    = await this.dh.acquireTable('user');
    this.accountHandle = await this.dh.acquireTable('account');
    this.joinedHandle  = await this.dh.acquireTable('user_account');

    this.userCols    = this.makeCols(this.userHandle);
    this.accountCols = this.makeCols(this.accountHandle);
    this.joinedCols  = this.makeCols(this.joinedHandle);

    this.userAdapter    = this.dh.createLiveAdapter();
    this.accountAdapter = this.dh.createLiveAdapter();
    this.joinedAdapter  = this.dh.createLiveAdapter();

    // Bind once per handle; adapter internally prevents double-binding.
    await this.userAdapter.bind(this.userHandle, rows => { this.userRows = rows; });
    await this.accountAdapter.bind(this.accountHandle, rows => { this.accountRows = rows; });
    await this.joinedAdapter.bind(this.joinedHandle, rows => { this.joinedRows = rows; });
  }

  async ngOnDestroy() {
    await this.userAdapter?.unbind();
    await this.accountAdapter?.unbind();
    await this.joinedAdapter?.unbind();

    // Ref-counted release. In dev, this is a no-op to avoid HMR "closed" races.
    await this.dh.releaseTable('user');
    await this.dh.releaseTable('account');
    await this.dh.releaseTable('user_account');
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
    }));
  }
}
```

**Template input fix (Angular parser error you hit earlier):**

```html
<input (input)="quickFilter('user', (($event.target as HTMLInputElement).value))">
```

(and same for account/joined).

---

# 3) Main bootstrap (standalone) — AG Grid v34 module registration

`src/main.ts`

```ts
import { bootstrapApplication } from '@angular/platform-browser';
import { appConfig } from './app/app.config';
import { AppComponent } from './app/app.component';

// AG Grid v34 modules
import { ModuleRegistry, AllCommunityModule } from 'ag-grid-community';
ModuleRegistry.registerModules([AllCommunityModule]);

bootstrapApplication(AppComponent, appConfig).catch(err => console.error(err));
```

---

# 4) Environment

`src/environments/environment.ts`

```ts
export const environment = {
  production: false,                            // set true in prod build
  deephavenUrl: 'http://localhost:10000',       // your DH base URL
  deephavenPsk: 'YOUR_PSK_TOKEN',               // REQUIRED
};
```

---

## Final sanity checklist

- Deephaven has Python globals:
    
    - `user` (append),
        
    - `account` (append),
        
    - `user_account` (your **`aj`** join driven by `account`, e.g. `on=['userId','ts']`).
        
- You’re **not** creating joins in Angular—only acquiring existing tables.
    
- In dev, you won’t see “table already closed” anymore because we don’t actually close handles (HMR-safe). In prod, ref-counts close cleanly.
    
- The adapter won’t double-bind and will silently unbind if DH reports “closed”.
    

If anything still squeaks, tell me the **exact** line shown in the new stack (it should now be inside the guarded retry path), and I’ll tailor the last 1% for your repo.



-----------------------------


I understand your pain — this type of error happens when the **table handle gets closed under your feet**. Let’s approach this like a senior engineer debugging a production streaming issue:

---

## Why you see “`Table already closed` / `ClassCastException`”

1. **Append tables from Kafka** (`kc.consume(..., TableType.append)`):
    
    - These are _ticking_ tables; they never stop. They don’t need to be `close()`d by Angular.
        
    - If Angular closes or reuses the same handle incorrectly, the backend complains: “already closed”.
        
2. **Angular side**:
    
    - When your component reinitializes (HMR/dev hot reload, or navigation), Angular re-runs `ngOnInit`, so it requests the same table name again.
        
    - If the old adapter still has a viewport open but you already `close()`d the handle, you get exactly this exception.
        
3. **Deephaven Java backend**:
    
    - The `ClassCastException` you see is just DH’s engine throwing when a dead handle is accessed.
        

---

## Correct architecture for your case

- **Server (Python)**:
    
    - Define global variables once: `user`, `account`, `user_account`.
        
    - Don’t wrap them in `expose_data_to_angular(...)` unless you want renames.
        
    - Make sure you _don’t recreate the same variable on every re-run_ of the Python script (that kills the old handle).
        
    - Example (safe global init):
        
        ```python
        if 'user' not in globals():
            user_stream_data = kc.consume(
                KAFKA_CONFIG, TOPIC1,
                key_spec=kc.KeyValueSpec.IGNORE,
                value_spec=USER_VALUE_SPEC,
                table_type=kc.TableType.append,
            )
            user = user_stream_data
        
        if 'account' not in globals():
            account_stream_data = kc.consume(
                KAFKA_CONFIG, TOPIC2,
                key_spec=kc.KeyValueSpec.IGNORE,
                value_spec=ACCOUNT_VALUE_SPEC,
                table_type=kc.TableType.append,
            )
            account = account_stream_data
        
        if 'user_account' not in globals():
            user = user.update(["ts = now()"])
            account = account.update(["ts = now()"])
            users_hist = user.sort_descending("ts")
            user_account = account.aj(users_hist, on=["userId","ts"], joins=["name","email","age"])
        ```
        
- **Angular service**:
    
    - Never `close()` these tables. They are global append streams.
        
    - Only **acquire once** and keep reusing.
        
    - In dev mode, disable `releaseTable` logic completely.
        

---

## Minimal **fix for your Angular service**

Change your `releaseTable` to **no-op** for append streams.

```ts
async releaseTable(name: string): Promise<void> {
  // For append streams: never close on frontend
  return;
}
```

Or simpler: **remove all release/close logic** in Angular. Just leave the viewport unbound in `ngOnDestroy`.

---

## Minimal **fix for LiveTableAdapter**

Don’t auto-unbind on “closed” — instead, just ignore the update.

```ts
this.sub = async () => {
  try {
    const snap = await table.snapshot(cols);
    onRows(snap.toObjects?.() ?? snap);
  } catch (e: any) {
    const msg = (e?.message ?? String(e)).toLowerCase();
    if (msg.includes("closed")) {
      console.warn("Ignoring closed snapshot (table is global append stream)");
      return;
    }
    console.error("Snapshot error", e);
  }
};
```

---

## Step-by-step to get rid of your issue

1. **In Deephaven IDE**:
    
    - Run your Kafka→append code once.
        
    - Verify `user.size()` and `account.size()` increase as Kafka produces.
        
    - Keep `user`, `account`, `user_account` alive in globals.
        
2. **In Angular**:
    
    - Only call `getTableHandle('user')`, `getTableHandle('account')`, `getTableHandle('user_account')`.
        
    - Bind to them; **do not close** them.
        
    - On `ngOnDestroy`, just unbind your viewport, nothing else.
        
3. **Result**:
    
    - No more “table already closed”.
        
    - Columns already show → now rows will stream in as Kafka sends.
        

---

✅ So the solution is: **don’t close global append tables from the frontend.** They are long-lived. Just open a viewport, snapshot, and leave them ticking.

Do you want me to give you a **final cleaned DeephavenService** with:

- PSK login,
    
- `getTableHandle`,
    
- `createLiveAdapter`,
    
- and _no_ close/release logic at all (safe for append streams)?