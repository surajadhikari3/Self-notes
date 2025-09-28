
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