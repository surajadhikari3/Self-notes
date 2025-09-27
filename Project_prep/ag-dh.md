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