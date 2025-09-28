You’re hitting “Table already closed” because the server-side variable is being redefined (or the DH session recycles) while your Angular viewport is still using the old handle. So your first snapshot works, the server re-runs Python (or you switch tabs, etc.), DH closes the old handle, and the next “safeSnapshot” throws.

Fix: make the client **self-healing**. When DH says “closed”, automatically **re-acquire a fresh handle by name**, re-open the viewport, and resume streaming — no page reloads, no manual close.

Below is a drop-in update for your service + the small component change to pass a resolver.

# 1) `deephaven.service.ts` (PSK required + self-healing adapter)

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

  /** Get a handle to an existing globals() table. */
  async getTableHandle(name: string): Promise<DhTable> {
    if (!this.ready) await this.connect();
    if (!/^[A-Za-z_][A-Za-z0-9_]*$/.test(name)) {
      throw new Error(`Invalid table variable name: "${name}"`);
    }
    return this.ide.getTable(name);
  }

  /** Create a live adapter. Pass a resolver to auto-reacquire when DH closes a handle. */
  createLiveAdapter() {
    return new LiveTableAdapter(this.dh);
  }
}

/**
 * LiveTableAdapter (self-healing)
 * - Opens a viewport and snapshots to array for AG Grid.
 * - If DH says "table closed", uses the provided resolver() to reacquire a fresh handle,
 *   reopens the viewport, and continues without crashing.
 */
class LiveTableAdapter {
  private dh: any;
  private table?: any;
  private viewport?: any;
  private sub?: (e: any) => void;
  private destroyed = false;
  private rebinding = false;

  constructor(dh: any) { this.dh = dh; }

  /**
   * Bind to a table. Provide a resolver that returns a *fresh* handle when needed.
   * Example resolver: () => deephavenService.getTableHandle('user_account')
   */
  async bind(
    table: any,
    onRows: (rows: any[]) => void,
    columns?: string[],
    resolver?: () => Promise<any>
  ) {
    await this.unbind(); // serialize
    this.destroyed = false;
    this.table = table;

    const cols = columns ?? (table?.columns ?? []).map((c: any) => c.name);

    // open viewport first
    this.viewport = await this.table.setViewport(0, 10000, cols);

    // initial snapshot
    await this.snapshotTo(onRows, cols, resolver);

    // updates
    const EVENT = (this.dh?.Table?.EVENT_UPDATED ?? 'update');
    this.sub = async () => {
      if (this.destroyed) return;
      await this.snapshotTo(onRows, cols, resolver);
    };
    this.table.addEventListener(EVENT, this.sub);
  }

  private async snapshotTo(
    onRows: (rows: any[]) => void,
    cols: string[],
    resolver?: () => Promise<any>
  ) {
    try {
      const snap = await this.table!.snapshot(cols);
      const rows = snap.toObjects?.() ?? snap;
      onRows(rows);
    } catch (e: any) {
      const msg = (e?.message ?? String(e)).toLowerCase();
      const isClosed = msg.includes('closed');
      if (!isClosed || !resolver || this.rebinding || this.destroyed) {
        // ignore non-closed errors / no resolver / already rebinding / destroyed
        return;
      }
      // self-heal: reacquire and rebind viewport + snapshot
      this.rebinding = true;
      try {
        const fresh = await resolver();
        if (this.destroyed) return;
        // detach old listener
        try {
          if (this.sub && this.table) {
            const EVENT = (this.dh?.Table?.EVENT_UPDATED ?? 'update');
            this.table.removeEventListener?.(EVENT, this.sub);
          }
        } catch {}
        // switch table
        this.table = fresh;
        // reopen viewport
        try { await this.viewport?.close?.(); } catch {}
        this.viewport = await this.table.setViewport(0, 10000, cols);

        // reattach listener
        const EVENT = (this.dh?.Table?.EVENT_UPDATED ?? 'update');
        this.sub = async () => {
          if (this.destroyed) return;
          await this.snapshotTo(onRows, cols, resolver);
        };
        this.table.addEventListener(EVENT, this.sub);

        // take snapshot now
        const snap = await this.table.snapshot(cols);
        const rows = snap.toObjects?.() ?? snap;
        onRows(rows);
      } finally {
        this.rebinding = false;
      }
    }
  }

  async unbind() {
    this.destroyed = true;
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

# 2) Component: pass a resolver so the adapter can reacquire

Update how you bind each grid — pass a lambda that fetches a **fresh** handle by name. You don’t need any close/release logic at all.

```ts
// join-tables.component.ts (relevant bits)
async ngOnInit() {
  await this.dh.connect();

  this.userHandle    = await this.dh.getTableHandle('user');
  this.accountHandle = await this.dh.getTableHandle('account');
  this.joinedHandle  = await this.dh.getTableHandle('user_account');

  this.userCols    = this.makeCols(this.userHandle);
  this.accountCols = this.makeCols(this.accountHandle);
  this.joinedCols  = this.makeCols(this.joinedHandle);

  this.userAdapter    = this.dh.createLiveAdapter();
  this.accountAdapter = this.dh.createLiveAdapter();
  this.joinedAdapter  = this.dh.createLiveAdapter();

  await this.userAdapter.bind(
    this.userHandle,
    rows => { this.userRows = rows; },
    undefined,
    () => this.dh.getTableHandle('user')               // <-- resolver
  );
  await this.accountAdapter.bind(
    this.accountHandle,
    rows => { this.accountRows = rows; },
    undefined,
    () => this.dh.getTableHandle('account')            // <-- resolver
  );
  await this.joinedAdapter.bind(
    this.joinedHandle,
    rows => { this.joinedRows = rows; },
    undefined,
    () => this.dh.getTableHandle('user_account')       // <-- resolver
  );
}

async ngOnDestroy() {
  await this.userAdapter?.unbind();
  await this.accountAdapter?.unbind();
  await this.joinedAdapter?.unbind();
  // no close/release calls; these are long-lived server tables
}
```

# 3) Deephaven (server) reminders

- Define `user`, `account`, `user_account` once and keep them alive.
    
- If you re-run the Python cell, the adapter will now auto-rebind using the resolver, so the UI won’t crash.
    
- For your append-only requirement, `aj` with `on=["userId","ts"]` is correct, where `ts` exists on both sides.
    

---

This version fixes the exception you’re seeing because, when the **safeSnapshot** hits a closed handle, it **reacquires a new handle by name** and **reopens the viewport** automatically. No more “already closed” loops, and no more dead grids.


------------------

You’re right: you should be able to open **one viewport** and let it stream forever.  
The reason you kept getting “table already closed” was not the viewport—it was the **server variable being replaced** when your Python cell re-ran (or when you reassigned `user = user.update(...)` again). When a globals() variable is **rebound**, Deephaven **closes** the old handle. Your Angular viewport was still attached to that old handle, so the next snapshot exploded.

Fix the root cause: **make the server tables stable** (defined once, never rebound), then use a **single long-lived viewport** in Angular (no reopen, no close).

---

# ✅ Deephaven (server) — define tables **once**

Put this in one cell (or a file) and run it once. It guards with a flag so re-running won’t rebind/close anything.

```python
from deephaven.stream.kafka import consumer as kc
from deephaven import dtypes as dt

if 'PIPELINE_READY' not in globals():

    # ---- Kafka config (yours here) ----
    KAFKA_CONFIG = {
        # ...
    }
    TOPIC1 = "user-topic"
    TOPIC2 = "account-topic"

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

    # ---- Consume append tables (exported as globals) ----
    user_stream_data = kc.consume(
        KAFKA_CONFIG, TOPIC1,
        key_spec=kc.KeyValueSpec.IGNORE,
        value_spec=USER_VALUE_SPEC,
        table_type=kc.TableType.append,
    )
    account_stream_data = kc.consume(
        KAFKA_CONFIG, TOPIC2,
        key_spec=kc.KeyValueSpec.IGNORE,
        value_spec=ACCOUNT_VALUE_SPEC,
        table_type=kc.TableType.append,
    )

    # IMPORTANT: assign once; do NOT reassign these variables later
    user = user_stream_data.table.update(["ts = now()"])
    account = account_stream_data.table.update(["ts = now()"])

    # user “dimension” history (latest-by time helps as-of semantics)
    users_hist = user.sort_descending("ts")

    # Account-driven append result; shows a new row only when *account* arrives,
    # and enriches with user fields as-of that time.
    user_account = account.aj(users_hist, on=["userId", "ts"], joins=["name","email","age"])

    PIPELINE_READY = True  # <-- guard flag

# From here, DO NOT reassign user/account/user_account variable names.
# If you need to change the pipeline, restart the DH console/kernel.
```

**Key rule:** after this is set, **do not run** lines like `user = user.update(...)` again. That rebinds the variable and implicitly closes the previous handle, killing any existing client viewports.

---

# ✅ Angular — single-viewport, no reopen/close

This service opens **one** viewport and never closes it unless you navigate away. No auto-reacquire, no release/close logic. PSK required.

`src/app/deephaven/deephaven.service.ts`

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

  get isReady() { return this.ready; }

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

  /** Get a handle to an existing globals() table (user / account / user_account). */
  async getTableHandle(name: string): Promise<DhTable> {
    if (!this.ready) await this.connect();
    if (!/^[A-Za-z_][A-Za-z0-9_]*$/.test(name)) {
      throw new Error(`Invalid table variable name: "${name}"`);
    }
    return this.ide.getTable(name);
  }

  /** One-viewport streaming adapter (no reopen/close). */
  createLiveAdapter() {
    return new LiveViewportAdapter(this.dh);
  }
}

/** Single viewport that streams updates; never closes unless you unbind() manually. */
class LiveViewportAdapter {
  private dh: any;
  private table?: any;
  private viewport?: any;
  private sub?: (e: any) => void;
  private bound = false;

  constructor(dh: any) {
    this.dh = dh;
  }

  async bind(table: any, onRows: (rows: any[]) => void, columns?: string[]) {
    if (this.bound) return;                 // bind only once
    this.bound = true;
    this.table = table;

    const cols = columns ?? (table?.columns ?? []).map((c: any) => c.name);

    // Open one viewport for the session and keep it
    this.viewport = await table.setViewport(0, 10000, cols);

    // Initial snapshot
    const snap = await table.snapshot(cols);
    onRows(snap.toObjects?.() ?? snap);

    // On each update, take a fresh snapshot (simple + robust)
    const EVENT = (this.dh?.Table?.EVENT_UPDATED ?? 'update');
    this.sub = async () => {
      try {
        const s = await table.snapshot(cols);
        onRows(s.toObjects?.() ?? s);
      } catch (e: any) {
        // If server-side variable was replaced, snapshot would fail.
        // With the guarded DH code above, that should no longer happen.
        console.error('Snapshot failed:', e?.message ?? e);
      }
    };
    table.addEventListener(EVENT, this.sub);
  }

  /** Optional cleanup when you navigate away. */
  async unbind() {
    if (!this.bound) return;
    this.bound = false;
    try {
      if (this.table && this.sub) {
        const EVENT = (this.dh?.Table?.EVENT_UPDATED ?? 'update');
        this.table.removeEventListener?.(EVENT, this.sub);
      }
    } catch {}
    // NOTE: we intentionally do NOT close the viewport or table here.
    // These are long-lived append streams; closing invites race conditions in dev.
    this.sub = undefined;
    // this.viewport = undefined; // keep a reference if you like; we don't close it
    // this.table = undefined;
  }
}
```

**Component usage stays the same** (bind once per table; no closing):

```ts
await this.dh.connect();

this.userHandle    = await this.dh.getTableHandle('user');
this.accountHandle = await this.dh.getTableHandle('account');
this.joinedHandle  = await this.dh.getTableHandle('user_account');

this.userCols    = this.makeCols(this.userHandle);
this.accountCols = this.makeCols(this.accountHandle);
this.joinedCols  = this.makeCols(this.joinedHandle);

this.userAdapter    = this.dh.createLiveAdapter();
this.accountAdapter = this.dh.createLiveAdapter();
this.joinedAdapter  = this.dh.createLiveAdapter();

await this.userAdapter.bind(this.userHandle,    rows => { this.userRows   = rows; });
await this.accountAdapter.bind(this.accountHandle, rows => { this.accountRows = rows; });
await this.joinedAdapter.bind(this.joinedHandle,  rows => { this.joinedRows  = rows; });

// optional onDestroy: await adapter.unbind(); (no table/viewport close)
```

And keep your AG Grid v34 bindings:

- `[rowData]="...Rows"`
    
- `api.setGridOption('quickFilterText', text)`
    

---

## TL;DR

- The **only** reason you saw openings/closings was **server rebinds**.
    
- Guard your Python so `user`, `account`, `user_account` are created **once** and **never reassigned**.
    
- On the client, open **one viewport** and keep it; **do not** close tables or recreate viewports while developing.
    

Do the two changes above and the stream will stay stable on the same viewport/port with continuous updates.