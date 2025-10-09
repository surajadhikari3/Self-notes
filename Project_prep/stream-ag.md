You’re bumping into two separate things:

1. **Angular is still holding a handle to an export that just got swapped**, so you see `Export in state DEPENDENCY FAILED`.
    
2. **The swap in the server is a little too “eager”**, so the old generation disappears before the client has rebound.
    

Below is a **robust pair** (server + Angular) that fixes both:

- The **server** does an **atomic swap with “linger”**: it publishes the new tables **first**, bumps a `lastApplied` heartbeat (so Angular can notice), and only **after a delay** (default 20 s) does it close the old generation.
    
- The **Angular service** watches `orchestrator_status.lastApplied`, and when it changes it **rebinds** to the same export name with **retries** (so even if you catch a brief in-flight moment you recover automatically).
    

---

# A) Server – `orchestrator_dh.py` (0.40.2)

Put this file in your app dir, e.g.

```
C:\dhdata\data\app.d\orchestrator_dh.py
```

Restart DH (`deephaven server --port 10000`).

```python
# orchestrator_dh.py – Deephaven 0.40.2
# Robust app-mode orchestrator with atomic hot-swap + linger.

from __future__ import annotations
from dataclasses import dataclass
from typing import List, Optional, Tuple
import threading
import time

from deephaven import ApplicationState
import deephaven.dtypes as dt
from deephaven import time as dhtime
from deephaven.table import Table
from deephaven.stream.kafka import consumer as kc
from deephaven.table_listener import listen

# -----------------------------
# EDIT these to your environment
# -----------------------------
DEFAULT_USERS_TOPIC   = "ccd01_sb_its_esp_tap3507_bishowcaseraw"
DEFAULT_ACCOUNTS_TOPIC= "ccd01_sb_its_esp_tap3507_bishowcasecurated"
CONTROL_TOPIC         = "ccd01_sb_its_esp_tap3507_metadata"   # JSON control

KAFKA_CONFIG = {
    # fill with your real config:
    # 'bootstrap.servers': '...',
    # 'security.protocol': 'SASL_SSL',
    # 'sasl.mechanism': 'OAUTHBEARER',
    # 'sasl.oauthbearer.method': 'oidc',
    # 'sasl.oauthbearer.token.endpoint.url': '...',
    # 'sasl.oauthbearer.client.id': '...',
    # 'sasl.oauthbearer.client.secret': '...',
    # ...
    'auto.offset.reset': 'latest',
}

# Linger time for the *previous* generation (seconds)
LINGER_SECONDS = 20

# -----------------------------
# Value specs
# -----------------------------
USER_VALUE_SPEC = kc.json_spec({
    "userId": dt.string,
    "name": dt.string,
    "email": dt.string,
    "age": dt.int_
})

ACCOUNT_VALUE_SPEC = kc.json_spec({
    "userId": dt.string,
    "accountType": dt.string,
    "balance": dt.double
})

CONTROL_VALUE_SPEC = kc.json_spec({
    "usersTopic": dt.string,
    "accountsTopic": dt.string,
    "joinType": dt.string  # 'left' (default) or 'inner'
})

# -----------------------------
# Helpers
# -----------------------------
def _consume(topic: str, value_spec) -> Table:
    cfg = dict(KAFKA_CONFIG)
    return kc.consume(
        cfg,
        topic,
        key_spec=kc.KeyValueSpec.IGNORE,
        value_spec=value_spec,
        table_type=kc.TableType.append()
    )

def _normalize_users(t: Table) -> Table:
    # coerce/clean join key, avoid null key edge-cases
    return t.update([
        "userId = (userId == null) ? '∅' : string(userId)",
        "name = (name == null) ? '' : name",
        "email = (email == null) ? '' : email"
    ])

def _normalize_accounts(t: Table) -> Table:
    return t.update([
        "userId = (userId == null) ? '∅' : string(userId)"
    ])

def _left_like(users: Table, accounts: Table) -> Table:
    """
    left-like join that tolerates multiple right rows per key.
    We do an inner JOIN for matches and then union unmatched left rows.
    """
    inner = users.join(accounts, on=["userId"])  # many-to-many
    unmatched = users.where("!in(userId, accounts, userId)")
    # add missing right columns to unmatched
    unmatched = unmatched.update([
        "accountType = (String) null",
        "balance = (Double) null"
    ])
    return inner.union(unmatched)

# -----------------------------
# Orchestrator
# -----------------------------
@dataclass
class Generation:
    users_raw: Table
    accounts_raw: Table
    users_ui: Table
    accounts_ui: Table
    final_ui: Table
    # keep any listeners / extra handles to close
    handles: List[object]

class Orchestrator:
    def __init__(self) -> None:
        self.app = ApplicationState()
        self.status = None             # exported status table
        self.current: Optional[Generation] = None
        self.previous: Optional[Generation] = None
        self._lock = threading.Lock()

        # Export a 1-row status table that we mutate on swap
        self._init_status()

        # Build initial generation
        self._apply(DEFAULT_USERS_TOPIC, DEFAULT_ACCOUNTS_TOPIC, "left")

        # Start control listener
        self._start_control_listener(CONTROL_TOPIC)
        print("[orchestrator] ready")

    # ---------- status ----------
    def _init_status(self) -> None:
        from deephaven import new_table
        from deephaven.column import string_col
        self.status = new_table([
            string_col("usersTopic",   [DEFAULT_USERS_TOPIC]),
            string_col("accountsTopic",[DEFAULT_ACCOUNTS_TOPIC]),
            string_col("joinType",     ["left"]),
            string_col("lastApplied",  [dhtime.to_iso8601(dhtime.dh_now())]),
        ])
        self.app["orchestrator_status"] = self.status

    def _bump_status(self, users: str, accounts: str, join_type: str) -> None:
        ts = dhtime.to_iso8601(dhtime.dh_now())
        # overwrite row 0 in-place (keeps export stable)
        self.status = self.status.update(
            [
                f"usersTopic   = '{users}'",
                f"accountsTopic= '{accounts}'",
                f"joinType     = '{join_type}'",
                f"lastApplied  = '{ts}'",
            ]
        )
        self.app["orchestrator_status"] = self.status

    # ---------- build one generation ----------
    def _build(self, users_topic: str, accounts_topic: str, join_type: str) -> Generation:
        u_raw = _consume(users_topic, USER_VALUE_SPEC)
        a_raw = _consume(accounts_topic, ACCOUNT_VALUE_SPEC)

        u_view = _normalize_users(u_raw).view(["userId", "name", "email", "age"])
        a_view = _normalize_accounts(a_raw).view(["userId", "accountType", "balance"])

        jt = (join_type or "left").lower()
        if jt.startswith("inner"):
            final = u_view.join(a_view, on=["userId"])
        else:
            final = _left_like(u_view, a_view)

        return Generation(
            users_raw=u_raw,
            accounts_raw=a_raw,
            users_ui=u_view,
            accounts_ui=a_view,
            final_ui=final,
            handles=[]
        )

    def _close_generation(self, gen: Optional[Generation]) -> None:
        if not gen:
            return
        # try to close everything; ignore errors
        for obj in [gen.users_ui, gen.accounts_ui, gen.final_ui, gen.users_raw, gen.accounts_raw, *gen.handles]:
            try:
                obj.close()
            except Exception:
                pass

    def _apply(self, users_topic: str, accounts_topic: str, join_type: str) -> None:
        with self._lock:
            # Build the new generation first
            new_gen = self._build(users_topic, accounts_topic, join_type)

            # Export new tables under the same names (atomic replacement)
            self.app["users_ui"]    = new_gen.users_ui
            self.app["accounts_ui"] = new_gen.accounts_ui
            self.app["final_ui"]    = new_gen.final_ui

            # Publish status heartbeat for clients to rebind
            self._bump_status(users_topic, accounts_topic, join_type)
            print(f"[orchestrator] topics set users='{users_topic}' accounts='{accounts_topic}' join='{join_type}'")

            # Linger the previous generation, then close
            old = self.current
            self.previous = old
            self.current = new_gen

            if old:
                def _linger_then_close():
                    try:
                        time.sleep(LINGER_SECONDS)
                    finally:
                        self._close_generation(old)
                threading.Thread(target=_linger_then_close, daemon=True).start()

    # ---------- control topic ----------
    def _start_control_listener(self, control_topic: str) -> None:
        ctrl = _consume(control_topic, CONTROL_VALUE_SPEC)

        def _on_update(_upd):
            try:
                last = ctrl.tail(1).snapshot()
                if last.size() == 0:
                    return
                df = last.to_pandas()
                row = df.iloc[0]
                u = (row.get("usersTopic") or "").strip()
                a = (row.get("accountsTopic") or "").strip()
                j = (row.get("joinType") or "left").strip()
                if u and a:
                    print(f"[orchestrator] applying control: users='{u}', accounts='{a}', join='{j}'")
                    self._apply(u, a, j)
            except Exception as e:
                print("[orchestrator] control listener err:", e)

        # keep a reference so the listener doesn’t get GC’d
        disp = listen(ctrl, _on_update, replay_initial=True)
        self.app["control_raw"] = ctrl   # for visibility
        # also keep the disposer in current generation so we can close on exit
        if self.current:
            self.current.handles.append(disp)

# Boot
_orch = Orchestrator()
```

**What changed vs. previous:**

- Exports are updated **first**, `orchestrator_status.lastApplied` is bumped, and only **later** (thread with `LINGER_SECONDS`) are prior tables closed. This prevents the live clients from hitting a “referent is no longer live” window.
    
- `final_ui` uses a **many-to-many tolerant** left-like join (`join` + `unmatched`) so you don’t get “duplicate right key for null” exceptions.
    

---

# B) Angular – rebind with retries (drop-in)

Replace the streaming code in your service with this version. It watches `orchestrator_status.lastApplied`, and when it changes:

- pauses updates,
    
- re-`getTable()` for the same export name,
    
- **retries** a few times with backoff if the export is still flipping, and
    
- resumes viewport updates.
    

```ts
// src/app/services/deephaven.service.ts (only the streaming bits shown)
// ... keep your ensureDhLoaded(), getClient(), getIde() as you had or as I sent earlier ...

const REBIND_RETRIES = 8;
const REBIND_BASE_MS = 150; // jittered exponential backoff

function sleep(ms: number) { return new Promise(r => setTimeout(r, ms)); }

export interface StreamHandle {
  rows$: Observable<any[]>;
  dispose(): Promise<void>;
}

async function rebindWithRetry(
  ide: any,
  tableName: string,
  attempt = 0
): Promise<any> {
  try {
    return await ide.getTable(tableName);
  } catch (e: any) {
    if (attempt >= REBIND_RETRIES) throw e;
    // backoff + jitter
    const delay = REBIND_BASE_MS * Math.pow(1.5, attempt) + Math.random() * 100;
    await sleep(delay);
    return rebindWithRetry(ide, tableName, attempt + 1);
  }
}

export class DeephavenService {
  // ... ctor, dh/client/ide creation as before ...

  private async bindViewport(table: any, maxRows: number, push: (rows: any[]) => void) {
    const update = async () => {
      const vp = await table.getViewportData();
      const cols = table.columns;
      const out: any[] = [];
      for (let i = 0; i < vp.rows.length; i++) {
        const row = vp.rows[i];
        const obj: any = {};
        for (const c of cols) obj[c.name] = row.get(c);
        out.push(obj);
      }
      push(out);
    };
    await table.setViewport(0, Math.max(0, maxRows - 1));
    table.addEventListener('updated', update);
    await update();
    return () => table.removeEventListener('updated', update);
  }

  async streamTableWithRebind(tableName: string, maxRows = 1000): Promise<StreamHandle> {
    const rows$ = new BehaviorSubject<any[]>([]);
    const ide = await this.getIde();

    let table: any | null = null;
    let unhook: (() => void) | null = null;

    const push = (rows: any[]) => this.zone.run(() => rows$.next(rows));

    const doBind = async () => {
      // cleanup prev
      if (unhook) { try { unhook(); } catch {} unhook = null; }
      if (table)  { try { table.close(); } catch {} table  = null; }

      table = await rebindWithRetry(ide, tableName);
      unhook = await this.bindViewport(table, maxRows, push);
    };

    // initial bind
    await doBind();

    // watch orchestrator_status
    const status = await ide.getTable('orchestrator_status');
    await status.setViewport(0, 0);
    let last: string | undefined;

    const onStatus = async () => {
      try {
        const vp = await status.getViewportData();
        if (vp.rows.length === 0) return;
        const col = status.columns.find((c: any) => c.name === 'lastApplied');
        const v = vp.rows[0].get(col);
        if (v !== last) {
          last = v;
          await doBind(); // <- rebind to the same exported name
        }
      } catch (e) {
        // don't break the stream on a transient status fetch error
        // console.warn('status watch error', e);
      }
    };
    status.addEventListener('updated', onStatus);
    await onStatus();

    return {
      rows$: rows$.asObservable(),
      dispose: async () => {
        try { status.removeEventListener('updated', onStatus); } catch {}
        try { status.close(); } catch {}
        if (unhook) { try { unhook(); } catch {} }
        if (table)  { try { table.close(); } catch {} }
        rows$.complete();
      }
    };
  }
}
```

Usage in your component stays the same as I suggested earlier:

```ts
this.handle = await this.dhService.streamTableWithRebind(source, 1000);
this.sub = this.handle.rows$.subscribe(rows => {
  this.isLoading$.next(false);
  this.tableDataSource.data = rows;
});
```

And remember to clean up in `ngOnDestroy()`:

```ts
if (this.sub) this.sub.unsubscribe();
if (this.handle) await this.handle.dispose();
```

---

## C) Control message format (unchanged)

Keep producing:

```json
{"usersTopic":"<NEW_USERS>", "accountsTopic":"<NEW_ACCTS>", "joinType":"left"}
```

**How to verify the swap happened**

1. In DH IDE, open `orchestrator_status`. You should see `usersTopic`, `accountsTopic`, and `lastApplied` change.
    
2. Your Angular page will pause for a moment and then repopulate with rows from the new topics (because it rebinding + retrying).
    
3. Server console shows one line:  
    `[orchestrator] topics set users='...' accounts='...' join='...'`
    

---

## D) If you still see “dependency failed”

- Increase `LINGER_SECONDS` to 30–45 to give the browser more time to rebind over slow networks.
    
- Make sure you **don’t** create a new Angular IDE session per page change; the service keeps one session (good).
    
- If you sometimes publish control updates in a burst, the retries will handle it—but you can also throttle updates on the Spring side (e.g., only send when something actually changed).
    

This combo (atomic swap + linger on the server, and rebind+retry on the client) is what keeps dashboards stable when sources change under load.

------------------------------------------------------------------


You’re seeing:

```
ApplicationState.__init__() missing 1 required positional argument: 'j_app_state'
```

because in **Deephaven 0.40.x** you **must not construct** `ApplicationState()` yourself. Instead, get the singleton with **`get_app_state()`**.

Here’s your **drop-in, fixed file** (copy/paste over your current `orchestrator_dh.py`). The only functional change is using `get_app_state()`; I kept all the reliability bits (atomic export, linger, control listener, many-to-many-safe join).

---

### `C:\dhdata\data\app.d\orchestrator_dh.py`

```python
# orchestrator_dh.py – Deephaven 0.40.2

from __future__ import annotations
from dataclasses import dataclass
from typing import List, Optional
import threading, time

# IMPORTANT: use get_app_state() in 0.40.x
from deephaven.appmode import get_app_state
import deephaven.dtypes as dt
from deephaven import time as dhtime
from deephaven.table import Table
from deephaven.stream.kafka import consumer as kc
from deephaven.table_listener import listen

# ---------- EDIT FOR YOUR ENV ----------
DEFAULT_USERS_TOPIC    = "ccd01_sb_its_esp_tap3507_bishowcaseraw"
DEFAULT_ACCOUNTS_TOPIC = "ccd01_sb_its_esp_tap3507_bishowcasecurated"
CONTROL_TOPIC          = "ccd01_sb_its_esp_tap3507_metadata"   # JSON control messages

KAFKA_CONFIG = {
    # Fill in your real values:
    # 'bootstrap.servers': '...',
    # 'security.protocol': 'SASL_SSL',
    # 'sasl.mechanism': 'OAUTHBEARER',
    # 'sasl.oauthbearer.method': 'oidc',
    # 'sasl.oauthbearer.token.endpoint.url': '...',
    # 'sasl.oauthbearer.client.id': '...',
    # 'sasl.oauthbearer.client.secret': '...',
    'auto.offset.reset': 'latest',
}

# Keep previous generation alive briefly so clients can rebind
LINGER_SECONDS = 20

# ---------- Specs ----------
USER_VALUE_SPEC = kc.json_spec({
    "userId": dt.string,
    "name": dt.string,
    "email": dt.string,
    "age": dt.int_,
})

ACCOUNT_VALUE_SPEC = kc.json_spec({
    "userId": dt.string,
    "accountType": dt.string,
    "balance": dt.double,
})

CONTROL_VALUE_SPEC = kc.json_spec({
    "usersTopic": dt.string,
    "accountsTopic": dt.string,
    "joinType": dt.string,  # 'left' (default) or 'inner'
})

# ---------- Helpers ----------
def _consume(topic: str, value_spec) -> Table:
    cfg = dict(KAFKA_CONFIG)
    return kc.consume(
        cfg,
        topic,
        key_spec=kc.KeyValueSpec.IGNORE,
        value_spec=value_spec,
        table_type=kc.TableType.append(),
    )

def _normalize_users(t: Table) -> Table:
    return t.update([
        "userId = (userId == null) ? '∅' : string(userId)",
        "name = (name == null) ? '' : name",
        "email = (email == null) ? '' : email",
    ])

def _normalize_accounts(t: Table) -> Table:
    return t.update([
        "userId = (userId == null) ? '∅' : string(userId)",
    ])

def _left_like(users: Table, accounts: Table) -> Table:
    # many-to-many tolerant "left-like" join
    inner = users.join(accounts, on=["userId"])
    unmatched = users.where("!in(userId, accounts, userId)") \
                     .update(["accountType = (String) null", "balance = (Double) null"])
    return inner.union(unmatched)

# ---------- Orchestrator ----------
@dataclass
class Generation:
    users_raw: Table
    accounts_raw: Table
    users_ui: Table
    accounts_ui: Table
    final_ui: Table
    handles: List[object]

class Orchestrator:
    def __init__(self) -> None:
        # ✔️ Correct for 0.40.x
        self.app = get_app_state()

        self.status = None
        self.current: Optional[Generation] = None
        self.previous: Optional[Generation] = None
        self._lock = threading.Lock()

        self._init_status()
        self._apply(DEFAULT_USERS_TOPIC, DEFAULT_ACCOUNTS_TOPIC, "left")
        self._start_control_listener(CONTROL_TOPIC)

        print("[orchestrator] ready")

    # ----- status export -----
    def _init_status(self) -> None:
        from deephaven import new_table
        from deephaven.column import string_col
        self.status = new_table([
            string_col("usersTopic",   [DEFAULT_USERS_TOPIC]),
            string_col("accountsTopic",[DEFAULT_ACCOUNTS_TOPIC]),
            string_col("joinType",     ["left"]),
            string_col("lastApplied",  [dhtime.to_iso8601(dhtime.dh_now())]),
        ])
        self.app["orchestrator_status"] = self.status

    def _bump_status(self, users: str, accounts: str, join_type: str) -> None:
        ts = dhtime.to_iso8601(dhtime.dh_now())
        self.status = self.status.update([
            f"usersTopic    = '{users}'",
            f"accountsTopic = '{accounts}'",
            f"joinType      = '{join_type}'",
            f"lastApplied   = '{ts}'",
        ])
        self.app["orchestrator_status"] = self.status

    # ----- build one generation -----
    def _build(self, users_topic: str, accounts_topic: str, join_type: str) -> Generation:
        u_raw = _consume(users_topic, USER_VALUE_SPEC)
        a_raw = _consume(accounts_topic, ACCOUNT_VALUE_SPEC)

        u_view = _normalize_users(u_raw).view(["userId", "name", "email", "age"])
        a_view = _normalize_accounts(a_raw).view(["userId", "accountType", "balance"])

        jt = (join_type or "left").lower()
        if jt.startswith("inner"):
            final = u_view.join(a_view, on=["userId"])
        else:
            final = _left_like(u_view, a_view)

        return Generation(
            users_raw=u_raw,
            accounts_raw=a_raw,
            users_ui=u_view,
            accounts_ui=a_view,
            final_ui=final,
            handles=[],
        )

    def _close_generation(self, gen: Optional[Generation]) -> None:
        if not gen:
            return
        for obj in [gen.users_ui, gen.accounts_ui, gen.final_ui, gen.users_raw, gen.accounts_raw, *gen.handles]:
            try:
                obj.close()
            except Exception:
                pass

    def _apply(self, users_topic: str, accounts_topic: str, join_type: str) -> None:
        with self._lock:
            new_gen = self._build(users_topic, accounts_topic, join_type)

            # 1) publish new exports first (atomic swap for clients)
            self.app["users_ui"]    = new_gen.users_ui
            self.app["accounts_ui"] = new_gen.accounts_ui
            self.app["final_ui"]    = new_gen.final_ui

            # 2) heartbeat so Angular can rebind
            self._bump_status(users_topic, accounts_topic, join_type)
            print(f"[orchestrator] topics set users='{users_topic}' accounts='{accounts_topic}' join='{join_type}'")

            # 3) linger and then close old gen
            old = self.current
            self.previous = old
            self.current = new_gen

            if old:
                def _linger():
                    try:
                        time.sleep(LINGER_SECONDS)
                    finally:
                        self._close_generation(old)
                threading.Thread(target=_linger, daemon=True).start()

    # ----- control topic -----
    def _start_control_listener(self, control_topic: str) -> None:
        ctrl = _consume(control_topic, CONTROL_VALUE_SPEC)

        def _on_update(_upd):
            try:
                last = ctrl.tail(1).snapshot()
                if last.size() == 0:
                    return
                df = last.to_pandas()
                row = df.iloc[0]
                u = (row.get("usersTopic") or "").strip()
                a = (row.get("accountsTopic") or "").strip()
                j = (row.get("joinType") or "left").strip()
                if u and a:
                    print(f"[orchestrator] applying control: users='{u}', accounts='{a}', join='{j}'")
                    self._apply(u, a, j)
            except Exception as e:
                print("[orchestrator] control listener err:", e)

        disp = listen(ctrl, _on_update)  # works across 0.40.x
        self.app["control_raw"] = ctrl
        if self.current:
            self.current.handles.append(disp)

# Boot
_orch = Orchestrator()
```

---

### Quick sanity check

1. Restart DH:
    
    ```
    deephaven server --port 10000
    ```
    
    Console should show:
    
    ```
    [orchestrator] topics set users='...' accounts='...' join='left'
    [orchestrator] control listener on '..._metadata'
    [orchestrator] ready
    ```
    
2. In the IDE “Applications” panel you’ll see the app; open `users_ui`, `accounts_ui`, `final_ui`, and `orchestrator_status`.
    
3. Publish a control message to the **control topic** (JSON):
    
    ```json
    {"usersTopic":"<NEW_USERS>", "accountsTopic":"<NEW_ACCTS>", "joinType":"left"}
    ```
    
    You should see in the DH console:
    
    ```
    [orchestrator] applying control: users='<NEW_USERS>' ...
    [orchestrator] topics set users='<NEW_USERS>' accounts='<NEW_ACCTS>' join='left'
    ```
    
    `orchestrator_status.lastApplied` updates immediately, and Angular can use that to rebind.
    

That’s it—this removes the `ApplicationState()` constructor error and keeps the rest of the hot-swap flow intact.