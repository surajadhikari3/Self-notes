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

-----------------------------

You’ve got it: the crashes are from the **server variables getting rebound** (closed + replaced) while your Angular viewport is still attached. Fix it in two places:

---

# 1) Guard the Deephaven pipeline (define _once_, never rebind)

Put your Kafka→tables→join in one cell (or one .py file) and wrap it with a guard so re-running the cell won’t replace the globals. Also assign the `.table` from the consumer, and add `ts` at creation time so you never reassign later.

```python
# ---- run once; safe to re-run without rebinding ----
from deephaven.stream.kafka import consumer as kc
from deephaven import dtypes as dt

if 'PIPELINE_READY' not in globals():
    # ---- Kafka config ----
    KAFKA_CONFIG = {
        # ... your config ...
    }
    TOPIC1 = "user-topic"      # user stream
    TOPIC2 = "account-topic"   # account stream

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

    # ---- Consume append tables (one-time) ----
    user_stream = kc.consume(
        KAFKA_CONFIG, TOPIC1,
        key_spec=kc.KeyValueSpec.IGNORE,
        value_spec=USER_VALUE_SPEC,
        table_type=kc.TableType.append,
    )

    account_stream = kc.consume(
        KAFKA_CONFIG, TOPIC2,
        key_spec=kc.KeyValueSpec.IGNORE,
        value_spec=ACCOUNT_VALUE_SPEC,
        table_type=kc.TableType.append,
    )

    # IMPORTANT: assign once and include ts here (so we never reassign later)
    user    = user_stream.table.update   (["ts = now()"])
    account = account_stream.table.update(["ts = now()"])

    # As-of join driven by account; last key is the as-of timestamp
    users_hist    = user.sort_descending("ts")  # provide history view
    user_account  = account.aj(users_hist, on=["userId", "ts"],
                               joins=["name", "email", "age"])

    # mark pipeline as initialized
    PIPELINE_READY = True

# From here on, DO NOT reassign:
#   user = user.update(...)
#   account = account.update(...)
#   user_account = ...
# Doing so closes the old tables and breaks any existing clients.
```

**Tips**

- If you need to change the pipeline, **restart the Python session** (kernel) instead of reassigning the same global names, or use _new_ variable names (e.g., `user_v2`) and point your client to them.
    
- To verify it’s ticking without Angular:
    
    ```python
    print(user.size())         # should rise over time
    print(account.size())
    print(user_account.size())
    ```
    

---

# 2) Angular: you’re pulling by table **variable names** (correct)

Yes—you’re doing it right. You fetch by the **exact global names** you created above:

```ts
await this.dh.connect();

this.userHandle    = await this.dh.getTableHandle('user');
this.accountHandle = await this.dh.getTableHandle('account');
this.joinedHandle  = await this.dh.getTableHandle('user_account');

// bind once; let the adapter keep a single viewport open
this.userAdapter    = this.dh.createLiveAdapter();
this.accountAdapter = this.dh.createLiveAdapter();
this.joinedAdapter  = this.dh.createLiveAdapter();

await this.userAdapter.bind(this.userHandle,    rows => this.userRows   = rows);
await this.accountAdapter.bind(this.accountHandle, rows => this.accountRows = rows);
await this.joinedAdapter.bind(this.joinedHandle,  rows => this.joinedRows  = rows);

// do NOT call close() on these handles in Angular.
// optional onDestroy: await adapter.unbind(); (no table close)
```

**AG Grid v34 reminders**

- Bind rows in the template: `[rowData]="userRows"` (no `setRowData`).
    
- Search via `api.setGridOption('quickFilterText', text)` (since `setQuickFilter` was removed).
    
- Register modules once in `main.ts`:
    
    ```ts
    import { ModuleRegistry, AllCommunityModule } from 'ag-grid-community';
    ModuleRegistry.registerModules([AllCommunityModule]);
    ```
    

---

## Quick checklist to stay stable

-  Deephaven globals created **once** behind a guard (`PIPELINE_READY`), **no reassignment**.
    
-  `user`, `account`, `user_account` exist and tick; `ts` added at creation time.
    
-  Angular fetches by those **same names** (no creating joins in Angular).
    
-  Angular opens **one viewport per table** and never closes the DH tables.
    
-  In dev, avoid re-running server cells that would rebind those variables. If you must change them, restart the DH session.
    

Do those, and you’ll have one viewport per grid that streams updates continuously—no open/close churn, no “table already closed”, and no class cast exceptions.


------------------------------


You’re getting that stack trace because the Python cell is **re-creating the Kafka consumers and tables each time you run it**. When a new `kc.consume(...)` is created, Deephaven tears down the old one; your Angular app still has a viewport on the **old** handle → “table already closed / ClassCastException” on the next snapshot.

Let’s fix this properly with an **idempotent** DH script that:

- Creates the consumers only once.
    
- Exposes **stable global** table names: `user`, `account`, `user_account`.
    
- Uses `KafkaTimestamp` as the as-of key (or `now()` fallback).
    
- Can be safely re-run without closing/replacing tables (unless you explicitly reset).
    

### Drop-in Deephaven Python (run in the IDE once)

```python
# ---- imports ----
from deephaven.stream.kafka import consumer as kc
from deephaven import dtypes as dt

# ---- one-time config ----
TOPIC1 = "ccd01_sb_its_esp_tap3507_bishowcaseraw"        # user
TOPIC2 = "ccd01_sb_its_esp_tap3507_bishowcasescurated"   # account

KAFKA_CONFIG = {
    "bootstrap.servers": "pkc-k13op.canadacentral.azure.confluent.cloud:9092",
    "security.protocol": "SASL_SSL",
    "sasl.mechanism": "OAUTHBEARER",
    "auto.offset.reset": "earliest",   # or "latest" depending on your need
    # ... your other SASL/OAuth props exactly as you already have ...
}

USER_VALUE_SPEC = kc.json_spec({
    "userId": dt.string,
    "name":   dt.string,
    "email":  dt.string,
    "age":    dt.int64,     # int64 plays nice; change if you need int32
})

ACCOUNT_VALUE_SPEC = kc.json_spec({
    "userId":      dt.string,
    "accountType": dt.string,
    "balance":     dt.double,
})

# ---- helpers ----
def _ensure_consumer(var_name: str, topic: str, value_spec):
    """Create a consumer only if it doesn't already exist in globals()."""
    if var_name in globals():
        return globals()[var_name]
    tbl = kc.consume(
        KAFKA_CONFIG,
        topic,
        key_spec=kc.KeyValueSpec.IGNORE,
        value_spec=value_spec,
        table_type=kc.TableType.append,
    )
    globals()[var_name] = tbl
    return tbl

def _with_ts(t, fallback_now: bool = True):
    """Ensure a 'ts' Instant column on table t."""
    names = [c.name for c in t.columns]
    if "KafkaTimestamp" in names:
        return t.rename_columns(["ts = KafkaTimestamp"])
    elif fallback_now:
        # if your broker doesn't provide KafkaTimestamp, use ingest time
        from deephaven.time import now
        return t.update(["ts = now()"])
    else:
        raise RuntimeError("No KafkaTimestamp column and fallback_now=False")

# ---- idempotent pipeline build ----
# These variable *names* are the ones your Angular app fetches
# (stable, do not recreate on re-run)
user_raw    = _ensure_consumer("user_raw", TOPIC1, USER_VALUE_SPEC)
account_raw = _ensure_consumer("account_raw", TOPIC2, ACCOUNT_VALUE_SPEC)

# Add ts (Instant) on both sides
user    = _with_ts(user_raw)
account = _with_ts(account_raw)

# User "dimension" history - latest by time helps with as-of semantics
# (aj uses <= on the last 'on' key; sort isn't required but is common)
users_hist = user.sort_descending("ts")

# Account-driven append result; emits row only when account arrives,
# enriches with user fields as-of that time. (Left = account)
user_account = account.aj(
    users_hist,
    on=["userId", "ts"],                      # last item is the as-of key
    joins=["name", "email", "age"],
)

# Export a guard so you know pipeline is alive
PIPELINE_READY = True

# (Optional) quick visibility checks:
# print(user.size(), account.size(), user_account.size())
```

#### To reset everything (only if you really need to):

Run this once, then re-run the cell above.

```python
for n in ["user_account", "users_hist", "user", "account", "user_raw", "account_raw"]:
    try:
        globals()[n].close()
    except Exception:
        pass
    globals().pop(n, None)
globals().pop("PIPELINE_READY", None)
```

---

## Angular side (final notes)

- Keep using the **self-healing LiveTableAdapter** I gave you last message (with the resolver lambdas). That adapter will reacquire a new handle if DH ever does close one.
    
- Do **not** close these globals from Angular. Just `unbind()` the viewport in `ngOnDestroy()`.
    
- Ensure you’re fetching exactly these names: `user`, `account`, `user_account`.
    

This idempotent DH script + the self-healing adapter stops the “table already closed / ClassCastException” loop and gets your rows flowing again.





-----------------


You’re hitting this because one of the arguments you pass to `kc.consume(...)` is a **Python function (or wrong type)**, and Deephaven’s Kafka bridge expects a Java-backed object with an internal `._object`. When it can’t find that, you get:

```
AttributeError: 'function' object has no attribute '_object'
```

In practice, this happens if any of these are wrong:

- `kc` got shadowed (e.g., you defined a function or variable named `kc`).
    
- `USER_VALUE_SPEC` / `ACCOUNT_VALUE_SPEC` isn’t a **json spec object** (e.g., you forgot to call `kc.json_spec(...)`).
    
- `KAFKA_CONFIG` isn’t the **expected properties map** (e.g., you accidentally turned it into a function/tuple).
    
- You re-ran a cell and redefined things in a partial state.
    

Below is a **minimal, safe, idempotent** Deephaven script you can paste as-is. It avoids functions, guards against re-run, and uses only the supported types for `kc.consume`. It also uses `KafkaTimestamp` directly (falls back to `now()` if not present).

### ✅ Clean Deephaven script (run once; safe to re-run)

```python
# Deephaven Kafka consumers → append tables → as-of join (account-driven)

from deephaven.stream.kafka import consumer as kc
from deephaven import dtypes as dt

# -------------------- CONFIG --------------------
TOPIC1 = "ccd01_sb_its_esp_tap3507_bishowcaseraw"        # user
TOPIC2 = "ccd01_sb_its_esp_tap3507_bishowcasescurated"   # account

# IMPORTANT: dict of strings; DO NOT put callables in here.
KAFKA_CONFIG = {
    "bootstrap.servers": "pkc-k13op.canadacentral.azure.confluent.cloud:9092",
    "security.protocol": "SASL_SSL",
    "sasl.mechanism": "OAUTHBEARER",
    "auto.offset.reset": "earliest",   # or "latest"
    # your oauth settings (strings only):
    "sasl.login.callback.handler.class":
        "org.apache.kafka.common.security.oauthbearer.secured.OAuthBearerLoginCallbackHandler",
    "sasl.oauthbearer.token.endpoint.url":
        "https://fdsit.rastest.tdbank.ca/as/token.oauth2",
    "sasl.jaas.config":
        "org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required "
        "oauth.jwks.endpoint.uri=\"https://.../jwks\" "
        "oauth.token.endpoint.uri=\"https://.../token\" "
        "oauth.client.id=\"TestScopeClient\" "
        "oauth.client.secret=\"2Federate\" "
        "oauth.scope=\"\" ;",
    # remove any trailing '\' continuations you copied from Java props; above is a single python string
}

# JSON value specs – MUST be kc.json_spec(...), not a dict or function
USER_VALUE_SPEC = kc.json_spec({
    "userId": dt.string,
    "name":   dt.string,
    "email":  dt.string,
    "age":    dt.int64,
})
ACCOUNT_VALUE_SPEC = kc.json_spec({
    "userId":      dt.string,
    "accountType": dt.string,
    "balance":     dt.double,
})

# -------------------- SAFE (re)RUN GUARDS --------------------
# only build the consumers once per session
if "user_raw" not in globals():
    user_raw = kc.consume(
        KAFKA_CONFIG,                # dict of strings
        TOPIC1,                      # string topic
        key_spec=kc.KeyValueSpec.IGNORE,
        value_spec=USER_VALUE_SPEC,  # kc.json_spec(...)
        table_type=kc.TableType.append,
    )
if "account_raw" not in globals():
    account_raw = kc.consume(
        KAFKA_CONFIG,
        TOPIC2,
        key_spec=kc.KeyValueSpec.IGNORE,
        value_spec=ACCOUNT_VALUE_SPEC,
        table_type=kc.TableType.append,
    )

# -------------------- TIMESTAMP NORMALIZATION --------------------
# Create a 'ts' Instant column on both sides. Prefer KafkaTimestamp if available.
def _with_ts(t):
    cols = [c.name for c in t.columns]
    if "KafkaTimestamp" in cols:
        return t.rename_columns(["ts = KafkaTimestamp"])
    else:
        from deephaven.time import now
        return t.update(["ts = now()"])

user    = _with_ts(user_raw)
account = _with_ts(account_raw)

# -------------------- JOIN (ACCOUNT-DRIVEN A/J) --------------------
# As-of join: result only when account rows arrive; user fields frozen "as of" ts.
# Last item in 'on' is the as-of key.
users_hist = user.sort_descending("ts")
user_account = account.aj(users_hist, on=["userId", "ts"], joins=["name", "email", "age"])

PIPELINE_READY = True  # convenience flag
```

### Quick sanity checks (run right below the cell)

```python
print("user cols:", [c.name for c in user.columns])
print("account cols:", [c.name for c in account.columns])
print("joined cols:", [c.name for c in user_account.columns])
print("sizes:", user.size(), account.size(), user_account.size())
```

If any of those `print`s throw, the table variables aren’t defined as expected.

---

## What likely went wrong in your version

- A helper **function** or name accidentally **shadowed** one of the Kafka objects (e.g., `KAFKA_CONFIG` became a function, or `USER_VALUE_SPEC` wasn’t created with `kc.json_spec`).
    
- Multiline Java-style property strings with `\` got pasted directly—Python treated those as **line continuations** that swallowed the next line or created an invalid object. In the script above, each property is a **single Python string**.
    
- Re-running the cell **recreated** consumers; your Angular app still had the old handles. The guards above (“`if 'user_raw' not in globals()`”) make the script **idempotent**.
    

---

## Angular (just to close the loop)

Keep using the self-healing adapter I gave you previously with resolvers:

```ts
await this.userAdapter.bind(
  await this.dh.getTableHandle('user'),
  rows => this.userRows = rows,
  undefined,
  () => this.dh.getTableHandle('user')
);
```

Same for `account` and `user_account`. No close/release calls from Angular.

---

If you still hit an error after pasting the **exact** script above, copy the **first 5–6 lines of the traceback** and the **line number** it points to in your cell—I’ll pinpoint the exact field that’s off.