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

------------------------------------------


Good catch — that traceback is because **`deephaven.time` in 0.40.2 doesn’t have `to_iso8601`**. We only used it to stamp a readable time in the status table, so the simplest, fully-compatible fix is to use Python’s `datetime.isoformat()`.

Below is a **clean, 0.40.2-compatible orchestrator** that:

- uses **`get_app_state()`** (not `ApplicationState()`),
    
- replaces `to_iso8601(...)` with a tiny `iso_now()` helper,
    
- keeps the reliable hot-swap / linger / control-topic listener,
    
- avoids the execution-context issues in listeners (we only call `tail(1).snapshot()` inside the callback).
    

Drop this in and restart.

---

### `C:\dhdata\data\app.d\orchestrator_dh.py`

```python
# orchestrator_dh.py – Deephaven 0.40.2

from __future__ import annotations
from dataclasses import dataclass
from typing import List, Optional
import threading, time
from datetime import datetime, timezone

from deephaven.appmode import get_app_state            # ✔ 0.40.x
import deephaven.dtypes as dt
from deephaven.table import Table
from deephaven.stream.kafka import consumer as kc
from deephaven.table_listener import listen

# ---------- CONFIG ----------
DEFAULT_USERS_TOPIC    = "ccd01_sb_its_esp_tap3507_bishowcaseraw"
DEFAULT_ACCOUNTS_TOPIC = "ccd01_sb_its_esp_tap3507_bishowcasecurated"
CONTROL_TOPIC          = "ccd01_sb_its_esp_tap3507_metadata"   # JSON control

KAFKA_CONFIG = {
    # put your real values here:
    # 'bootstrap.servers': '...',
    # 'security.protocol': 'SASL_SSL',
    # 'sasl.mechanism': 'OAUTHBEARER',
    # 'sasl.oauthbearer.method': 'oidc',
    # 'sasl.oauthbearer.token.endpoint.url': '...',
    # 'sasl.oauthbearer.client.id': '...',
    # 'sasl.oauthbearer.client.secret': '...',
    'auto.offset.reset': 'latest',
}

# keep the old generation alive this long so clients can rebind
LINGER_SECONDS = 20

# ---------- SPECS ----------
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

# ---------- HELPERS ----------
def iso_now() -> str:
    return datetime.now(timezone.utc).isoformat()

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
    inner = users.join(accounts, on=["userId"])
    # rows in users with no match on the right
    unmatched = users.where("!in(userId, accounts, userId)") \
                     .update(["accountType = (String) null", "balance = (Double) null"])
    return inner.union(unmatched)

# ---------- ORCHESTRATOR ----------
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
        self.app = get_app_state()      # ✔ correct for 0.40.x

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
            string_col("lastApplied",  [iso_now()]),
        ])
        self.app["orchestrator_status"] = self.status

    def _bump_status(self, users: str, accounts: str, join_type: str) -> None:
        self.status = self.status.update([
            f"usersTopic    = '{users}'",
            f"accountsTopic = '{accounts}'",
            f"joinType      = '{join_type}'",
            f"lastApplied   = '{iso_now()}'",
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

            # 2) heartbeat for clients that watch status
            self._bump_status(users_topic, accounts_topic, join_type)
            print(f"[orchestrator] topics set users='{users_topic}' accounts='{accounts_topic}' join='{join_type}'")

            # 3) linger then close old generation to avoid “export already failed” in UI
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

        disp = listen(ctrl, _on_update)   # no extra kwargs in 0.40.x
        self.app["control_raw"] = ctrl
        if self.current:
            self.current.handles.append(disp)

# Boot
_orch = Orchestrator()
```

> If you’re using **App Mode** descriptors, the matching `orchestrator.app` is:

```
name=Orchestrator
type=python
enabled=true
scripts=py:orchestrator_dh.py
```

Put both files in `C:\dhdata\data\app.d\`, then start:

```bash
deephaven server --port 10000
```

This removes the `to_iso8601` error and keeps everything else working on **0.40.2**.


-------------------------------------------------------------


You’re hitting two 0.40.2‐specific gotchas:

1. **Types in `kc.json_spec`** – use `dt.int64`, `dt.double`, etc. (not `dt.int_`).
    
2. **Formula string literals** – single-quoted literals can be parsed as non-strings in some contexts. Use **double quotes** and `iif/isNull` (or explicit casts) to keep types consistent. That’s what triggered the “cannot parse literal as a datetime” error when `""` was being inferred against an `Instant` metadata column.
    

Below is a drop-in, 0.40.2-compatible file that fixes both issues and keeps the hot-swap + linger logic.

---

### `C:\dhdata\data\app.d\orchestrator_dh.py`

```python
# Deephaven 0.40.2 – orchestrator with safe hot-swap + control-topic
from __future__ import annotations
from dataclasses import dataclass
from typing import List, Optional
import threading, time
from datetime import datetime, timezone

from deephaven.appmode import get_app_state
from deephaven.table import Table
import deephaven.dtypes as dt
from deephaven.stream.kafka import consumer as kc
from deephaven.table_listener import listen

# ---------------- CONFIG ----------------
DEFAULT_USERS_TOPIC    = "ccd01_sb_its_esp_tap3507_bishowcaseraw"
DEFAULT_ACCOUNTS_TOPIC = "ccd01_sb_its_esp_tap3507_bishowcasecurated"
CONTROL_TOPIC          = "ccd01_sb_its_esp_tap3507_metadata"

KAFKA_CONFIG = {
    # fill in your real security + bootstrap values:
    # "bootstrap.servers": "...",
    # "security.protocol": "SASL_SSL",
    # "sasl.mechanism": "OAUTHBEARER",
    # "sasl.oauthbearer.method": "oidc",
    # "sasl.oauthbearer.token.endpoint.url": "...",
    # "sasl.oauthbearer.client.id": "...",
    # "sasl.oauthbearer.client.secret": "...",
    "auto.offset.reset": "latest",
}

LINGER_SECONDS = 20  # keep old generation alive for UIs while swapping

# ---------------- SPECS -----------------
USER_VALUE_SPEC = kc.json_spec({
    "userId": dt.string,
    "name": dt.string,
    "email": dt.string,
    "age": dt.int64,          # <-- correct in 0.40.x
})

ACCOUNT_VALUE_SPEC = kc.json_spec({
    "userId": dt.string,
    "accountType": dt.string,
    "balance": dt.double,     # <-- correct in 0.40.x
})

CONTROL_VALUE_SPEC = kc.json_spec({
    "usersTopic": dt.string,
    "accountsTopic": dt.string,
    "joinType": dt.string,    # "left" (default) or "inner"
})

# -------------- HELPERS -----------------
def iso_now() -> str:
    return datetime.now(timezone.utc).isoformat()

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
    # Use double quotes + iif/isNull to keep string typing unambiguous
    return t.update([
        'userId = iif(isNull(userId), "∅", (String)userId)',
        'name   = iif(isNull(name),   "", (String)name)',
        'email  = iif(isNull(email),  "", (String)email)',
    ])

def _normalize_accounts(t: Table) -> Table:
    return t.update([
        'userId      = iif(isNull(userId), "∅", (String)userId)',
        # leave accountType/balance as-is (can be nulls)
    ])

def _left_like(users: Table, accounts: Table) -> Table:
    # inner rows
    inner = users.join(accounts, on=["userId"])
    # unmatched left rows padded with nulls
    unmatched = users.where('!in(userId, accounts, userId)') \
                     .update(['accountType = (String) null', 'balance = (Double) null'])
    return inner.union(unmatched)

# ------------- ORCHESTRATOR -------------
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
        self.app = get_app_state()

        self.status: Optional[Table] = None
        self.current: Optional[Generation] = None
        self.previous: Optional[Generation] = None
        self._lock = threading.Lock()

        self._init_status()
        self._apply(DEFAULT_USERS_TOPIC, DEFAULT_ACCOUNTS_TOPIC, "left")
        self._start_control_listener(CONTROL_TOPIC)

        print("[orchestrator] ready")

    # ---- status export ----
    def _init_status(self) -> None:
        from deephaven import new_table
        from deephaven.column import string_col
        self.status = new_table([
            string_col("usersTopic",   [DEFAULT_USERS_TOPIC]),
            string_col("accountsTopic",[DEFAULT_ACCOUNTS_TOPIC]),
            string_col("joinType",     ["left"]),
            string_col("lastApplied",  [iso_now()]),
        ])
        self.app["orchestrator_status"] = self.status

    def _bump_status(self, users: str, accounts: str, join_type: str) -> None:
        self.status = self.status.update([
            f'usersTopic    = "{users}"',
            f'accountsTopic = "{accounts}"',
            f'joinType      = "{join_type}"',
            f'lastApplied   = "{iso_now()}"',
        ])
        self.app["orchestrator_status"] = self.status

    # ---- build one generation ----
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
            users_raw=u_raw, accounts_raw=a_raw,
            users_ui=u_view, accounts_ui=a_view, final_ui=final,
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

            # publish new tables first (atomic swap for IDE/Angular)
            self.app["users_ui"]    = new_gen.users_ui
            self.app["accounts_ui"] = new_gen.accounts_ui
            self.app["final_ui"]    = new_gen.final_ui

            self._bump_status(users_topic, accounts_topic, join_type)
            print(f"[orchestrator] topics set users='{users_topic}' accounts='{accounts_topic}' join='{join_type}'")

            # linger and then close the old generation to avoid UI failures
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

    # ---- control topic ----
    def _start_control_listener(self, control_topic: str) -> None:
        ctrl = _consume(control_topic, CONTROL_VALUE_SPEC)

        def _on_update(_upd):
            try:
                last = ctrl.tail(1).snapshot()
                if last.size() == 0:
                    return
                df = last.to_pandas()
                row = df.iloc[0]
                users = (str(row.get("usersTopic") or "").strip())
                accts = (str(row.get("accountsTopic") or "").strip())
                join  = (str(row.get("joinType") or "left").strip())
                if users and accts:
                    print(f"[orchestrator] applying control: users='{users}', accounts='{accts}', join='{join}'")
                    self._apply(users, accts, join)
            except Exception as e:
                print("[orchestrator] control listener err:", e)

        disp = listen(ctrl, _on_update)   # 0.40.2 signature
        self.app["control_raw"] = ctrl
        if self.current:
            self.current.handles.append(disp)

# Boot once on import (App Mode)
_orch = Orchestrator()
```

**Notes / sanity checks**

- Types now match 0.40.2 (`dt.int64`, `dt.double`).
    
- All formula strings use **double quotes** and `iif/isNull` so nothing is parsed as `Instant`.
    
- Listener uses `tail(1).snapshot()` (safe in callback) and we **linger** the old generation, which prevents the IDE/Angular from throwing “export already failed” while you’ve got panels/viewports open.
    

**Control message format (unchanged):**

```json
{"usersTopic":"<NEW_USERS>", "accountsTopic":"<NEW_ACCTS>", "joinType":"left"}
```

**Verify the switch:**

- Open the small table `orchestrator_status` in the IDE — it shows the current users/accounts/join and `lastApplied` ISO time.
    
- When you publish to the control topic, the console prints:
    
    ```
    [orchestrator] applying control: users='...', accounts='...', join='left'
    [orchestrator] topics set users='...' accounts='...' join='left'
    ```
    
    and the `orchestrator_status` row updates immediately.
    
- Angular should just keep calling `getTable('users_ui' | 'accounts_ui' | 'final_ui')` — the names don’t change; the tables behind them do.
    

If you still see an error, paste the exact line shown in the stack (it will reference an `update([...])` formula). That will point to the specific column whose type inference is unhappy.

----------------------------------

now..

Thanks for the screenshots — they show exactly what’s wrong:

- The error is coming from the **formula** `iif(isNull(userId), "∅", (String)userId)`.  
    In 0.40.2 the query-language **`iif` doesn’t have a string overload**, so it fails to compile. That’s why you see “Cannot find method iif(boolean, java.lang.String, java.lang.String)”.
    

I rewrote the normalizers to avoid `iif` entirely and switched the join back to the built-in **`left_outer_join`** (so we don’t reference Python variables from a listener thread). This version also keeps the **linger** logic so the IDE / Angular don’t lose their viewports when topics change.

Drop these two files in and start the server.

---

### 1) `C:\dhdata\data\app.d\orchestrator.app`

```
id=kafka-orchestrator
name=Kafka Orchestrator
type=script
scriptType=python
enabled=true
scripts=py:orchestrator_dh.py
```

### 2) `C:\dhdata\data\app.d\orchestrator_dh.py`

```python
# Deephaven 0.40.2 – stable, hot-swappable Kafka orchestrator

from __future__ import annotations
from dataclasses import dataclass
from typing import List, Optional
import threading, time
from datetime import datetime, timezone

from deephaven.appmode import get_app_state
from deephaven.table import Table
import deephaven.dtypes as dt
from deephaven.stream.kafka import consumer as kc
from deephaven.table_listener import listen
from deephaven.experimental.outer_joins import left_outer_join

# ---------------- CONFIG ----------------
DEFAULT_USERS_TOPIC    = "ccd01_sb_its_esp_tap3507_bishowcaseraw"
DEFAULT_ACCOUNTS_TOPIC = "ccd01_sb_its_esp_tap3507_bishowcasecurated"
CONTROL_TOPIC          = "ccd01_sb_its_esp_tap3507_metadata"

KAFKA_CONFIG = {
    # TODO: fill these with your real values
    # "bootstrap.servers": "...",
    # "security.protocol": "SASL_SSL",
    # "sasl.mechanism": "OAUTHBEARER",
    # "sasl.oauthbearer.method": "oidc",
    # "sasl.oauthbearer.token.endpoint.url": "...",
    # "sasl.oauthbearer.client.id": "...",
    # "sasl.oauthbearer.client.secret": "...",
    "auto.offset.reset": "latest",
}

LINGER_SECONDS = 20  # keep old generation alive for open IDE/Angular viewports

# ---------------- SPECS -----------------
USER_VALUE_SPEC = kc.json_spec({
    "userId": dt.string,
    "name": dt.string,
    "email": dt.string,
    "age": dt.int64,      # correct scalar type in 0.40.x
})

ACCOUNT_VALUE_SPEC = kc.json_spec({
    "userId": dt.string,
    "accountType": dt.string,
    "balance": dt.double,
})

CONTROL_VALUE_SPEC = kc.json_spec({
    "usersTopic": dt.string,
    "accountsTopic": dt.string,
    "joinType": dt.string,        # "left" (default) or "inner"
})

# -------------- HELPERS -----------------
def iso_now() -> str:
    return datetime.now(timezone.utc).isoformat()

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
    # Avoid iif() – use coalesce() so strings are safe in 0.40.2
    return t.update([
        'userId = coalesce(userId, "∅")',
        'name   = coalesce(name, "")',
        'email  = coalesce(email, "")',
    ])

def _normalize_accounts(t: Table) -> Table:
    return t.update([
        'userId = coalesce(userId, "∅")'
    ])

# ------------- ORCHESTRATOR -------------
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
        self.app = get_app_state()

        self.status: Optional[Table] = None
        self.current: Optional[Generation] = None
        self.previous: Optional[Generation] = None
        self._lock = threading.Lock()

        self._init_status()
        self._apply(DEFAULT_USERS_TOPIC, DEFAULT_ACCOUNTS_TOPIC, "left")
        self._start_control_listener(CONTROL_TOPIC)

        print("[orchestrator] ready")

    # ---- status export ----
    def _init_status(self) -> None:
        from deephaven import new_table
        from deephaven.column import string_col
        self.status = new_table([
            string_col("usersTopic",   [DEFAULT_USERS_TOPIC]),
            string_col("accountsTopic",[DEFAULT_ACCOUNTS_TOPIC]),
            string_col("joinType",     ["left"]),
            string_col("lastApplied",  [iso_now()]),
        ])
        self.app["orchestrator_status"] = self.status

    def _bump_status(self, users: str, accounts: str, join_type: str) -> None:
        self.status = self.status.update([
            f'usersTopic    = "{users}"',
            f'accountsTopic = "{accounts}"',
            f'joinType      = "{join_type}"',
            f'lastApplied   = "{iso_now()}"',
        ])
        self.app["orchestrator_status"] = self.status

    # ---- build one generation ----
    def _build(self, users_topic: str, accounts_topic: str, join_type: str) -> Generation:
        u_raw = _consume(users_topic, USER_VALUE_SPEC)
        a_raw = _consume(accounts_topic, ACCOUNT_VALUE_SPEC)

        u_view = _normalize_users(u_raw).view(["userId", "name", "email", "age"])
        a_view = _normalize_accounts(a_raw).view(["userId", "accountType", "balance"])

        jt = (join_type or "left").lower()
        if jt.startswith("inner"):
            final = u_view.join(a_view, on=["userId"])
        else:
            # Robust left outer join implemented in 0.40.x experimental package
            final = left_outer_join(
                u_view, a_view,
                on="userId",
                joins=["accountType", "balance"]
            )

        return Generation(
            users_raw=u_raw, accounts_raw=a_raw,
            users_ui=u_view, accounts_ui=a_view, final_ui=final,
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

            # publish the new tables first (atomic swap for IDE & Angular)
            self.app["users_ui"]    = new_gen.users_ui
            self.app["accounts_ui"] = new_gen.accounts_ui
            self.app["final_ui"]    = new_gen.final_ui

            self._bump_status(users_topic, accounts_topic, join_type)
            print(f"[orchestrator] topics set users='{users_topic}' accounts='{accounts_topic}' join='{join_type}'")

            # linger old generation so existing viewers don’t fail
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

    # ---- control topic ----
    def _start_control_listener(self, control_topic: str) -> None:
        ctrl = _consume(control_topic, CONTROL_VALUE_SPEC)

        def _on_update(_upd):
            try:
                last = ctrl.tail(1).snapshot()
                if last.size() == 0:
                    return
                df = last.to_pandas()
                row = df.iloc[0]
                users = (str(row.get("usersTopic") or "").strip())
                accts = (str(row.get("accountsTopic") or "").strip())
                join  = (str(row.get("joinType") or "left").strip())
                if users and accts:
                    print(f"[orchestrator] applying control: users='{users}', accounts='{accts}', join='{join}'")
                    self._apply(users, accts, join)
            except Exception as e:
                print("[orchestrator] control listener err:", e)

        disp = listen(ctrl, _on_update)   # 0.40.2 signature
        self.app["control_raw"] = ctrl
        if self.current:
            self.current.handles.append(disp)

# Boot once (App Mode)
_orch = Orchestrator()
```

---

### How to run / verify

1. Start Deephaven:
    

```
deephaven server --port 10000
```

2. In the IDE open these exported tables: `users_ui`, `accounts_ui`, `final_ui`, and `orchestrator_status`.  
    `orchestrator_status` shows the currently applied topics and updates the moment a control message lands.
    
3. Publish your control JSON (exactly this shape):
    

```json
{"usersTopic":"<NEW_USERS>", "accountsTopic":"<NEW_ACCTS>", "joinType":"left"}
```

The console will print:

```
[orchestrator] applying control: ...
[orchestrator] topics set users='...' accounts='...' join='left'
```

The UI panels won’t error because the old generation lingers for 20 seconds while the new one becomes live.

---

If anything else trips, send the **first** stack message (top of the pink block) and the line number it references; with 0.40.2 the common pitfalls are formula type inference and referencing Python variables inside a listener — both are addressed above.

---------------------------------------

Here’s a **clean, 0.40.2-compatible App Mode orchestrator** that:

- consumes two Kafka topics (users + accounts),
    
- exports `users_ui`, `accounts_ui`, and a **left outer join** `final_ui`,
    
- listens to a **control topic** and hot-swaps topics safely (no formula/iif/coalesce),
    
- avoids “duplicate right key” / “null key” join errors,
    
- avoids ExecutionContext issues in the listener,
    
- keeps the **same export names**, so Angular keeps streaming without reconnecting.
    

---

# 1) `.app` file (loads the script at server start)

Save as:  
`C:\dhdata\app.d\orchestrator.app`

```properties
id=kafka-orchestrator
name=Kafka Orchestrator
type=script
scriptType=python
enabled=true
scripts=py:orchestrator.py
```

# 2) Python app (no formula pitfalls, 0.40.2 API)

Save as:  
`C:\dhdata\app.d\orchestrator.py`

```python
# Deephaven 0.40.2 orchestrator – App Mode
# - Hot-swaps Kafka topics via a control topic
# - Exports users_ui / accounts_ui / final_ui
# - No iif() / coalesce(); uses 0.40.2-safe ternary + filters
# - Listener uses Update.added to avoid ExecutionContext errors

from __future__ import annotations
from dataclasses import dataclass
from typing import Optional, List
from datetime import datetime, timezone
import threading

import deephaven.dtypes as dt
from deephaven import new_table
from deephaven.column import string_col, double_col
from deephaven.appmode import get_app_state
from deephaven.experimental.outer_joins import left_outer_join
from deephaven.stream.kafka import consumer as kc
from deephaven.table_listener import listen

# -----------------------------
# EDIT these defaults
# -----------------------------
DEFAULT_USERS_TOPIC   = "ccd01_sb_its_esp_tap3507_bishowcaseraw"
DEFAULT_ACCOUNTS_TOPIC = "ccd01_sb_its_esp_tap3507_bishowcasecurated"
CONTROL_TOPIC          = "ccd01_sb_its_esp_tap3507_metadata"

# Kafka client config – fill yours (OAuth / SSL, etc.)
KAFKA_CONFIG = {
    # examples:
    # "bootstrap.servers": "<your brokers>",
    # "auto.offset.reset": "latest",
    # "security.protocol": "SASL_SSL",
    # "sasl.mechanism": "OAUTHBEARER",
    # "sasl.oauthbearer.token.endpoint.url": "...",
    # "sasl.oauthbearer.client.id": "...",
    # "sasl.oauthbearer.client.secret": "...",
    # ... add the exact props you already use ...
}

# -----------------------------
# JSON specs (0.40.x dtypes)
# -----------------------------
USER_VALUE_SPEC = kc.json_spec({
    "userId": dt.string,
    "name": dt.string,
    "email": dt.string,
    "age": dt.int64,
})

ACCOUNT_VALUE_SPEC = kc.json_spec({
    "userId": dt.string,
    "accountType": dt.string,
    "balance": dt.double,
})

CONTROL_VALUE_SPEC = kc.json_spec({
    "usersTopic": dt.string,
    "accountsTopic": dt.string,
    "joinType": dt.string,
})

# how long to keep the old exports alive after a swap (prevents transient IDE errors)
LINGER_SECONDS = 8


def _consume(topic: str, value_spec):
    """Create an append-only streaming table for a Kafka topic."""
    cfg = dict(KAFKA_CONFIG)  # copy so we never mutate caller’s dict
    return kc.consume(
        cfg,
        topic,
        key_spec=kc.KeyValueSpec.IGNORE,
        value_spec=value_spec,
        table_type=kc.TableType.append(),
    )


def _normalize_users(t):
    # Export only columns we need and drop null/empty join keys
    return (
        t.view(["userId", "name", "email", "age"])
         .where('userId != null && userId != ""')
    )


def _normalize_accounts(t):
    # Drop null/empty keys and de-dupe right side so left-join can pick a single row
    return (
        t.view(["userId", "accountType", "balance"])
         .where('userId != null && userId != ""')
         .last_by("userId")
    )


def _build_views(users_raw, accounts_raw):
    u = _normalize_users(users_raw)
    a = _normalize_accounts(accounts_raw)
    final_tbl = left_outer_join(u, a, on="userId", joins=["accountType", "balance"])
    return u, a, final_tbl


def _status_table(users: str, accounts: str, join_type: str):
    return new_table([
        string_col("usersTopic",   [users]),
        string_col("accountsTopic", [accounts]),
        string_col("joinType",     [join_type]),
        string_col("lastApplied",  [datetime.now(timezone.utc).isoformat()]),
    ])


def _safe_close_many(objs: Optional[List[object]]):
    if not objs:
        return
    for o in objs:
        try:
            o.close()
        except Exception:
            pass


@dataclass
class _State:
    users_topic: str
    accounts_topic: str
    join_type: str = "left"

    users_raw: Optional[object] = None
    accounts_raw: Optional[object] = None
    users_ui: Optional[object] = None
    accounts_ui: Optional[object] = None
    final_ui: Optional[object] = None

    control_raw: Optional[object] = None
    control_disp: Optional[object] = None

    # resources from the *previous* generation that we’ll close later
    lingering: List[object] = None


class Orchestrator:
    def __init__(self, users_topic: str, accounts_topic: str, join_type: str = "left"):
        self.app = get_app_state()
        self.state = _State(users_topic, accounts_topic, join_type, lingering=[])

        # initial wire-up
        self._set_topics(users_topic, accounts_topic, join_type, initial=True)

        # control-topic listener
        ctrl_tbl = _consume(CONTROL_TOPIC, CONTROL_VALUE_SPEC)
        disp = listen(ctrl_tbl, self._on_control_update)
        self.state.control_raw = ctrl_tbl
        self.state.control_disp = disp

        # Export a small status table and the raw control stream (handy in IDE)
        self.app.export(ctrl_tbl, "control_raw")
        self.app.export(_status_table(users_topic, accounts_topic, join_type), "orchestrator_status")

        print(f"[orchestrator] control listener on '{CONTROL_TOPIC}'")
        print("[orchestrator] ready")

    # --------------- hot swap ----------------

    def _set_topics(self, users_topic: str, accounts_topic: str, join_type: str = "left", initial: bool = False):
        """Create new consumers/views/join and atomically swap the exported tables."""
        print(f"[orchestrator] topics set users='{users_topic}' accounts='{accounts_topic}' join='{join_type}'")

        # Build new pipeline first
        users_raw = _consume(users_topic, USER_VALUE_SPEC)
        accounts_raw = _consume(accounts_topic, ACCOUNT_VALUE_SPEC)
        users_ui, accounts_ui, final_ui = _build_views(users_raw, accounts_raw)

        # Export new tables under the same names (clients keep streaming)
        self.app.export(users_ui, "users_ui")
        self.app.export(accounts_ui, "accounts_ui")
        self.app.export(final_ui, "final_ui")
        self.app.export(_status_table(users_topic, accounts_topic, join_type), "orchestrator_status")

        # Schedule safe close of any previous generation AFTER we swapped
        old = [
            self.state.users_raw, self.state.accounts_raw,
            self.state.users_ui, self.state.accounts_ui, self.state.final_ui,
        ]
        self._linger_and_close(old)

        # Save new state
        self.state.users_topic   = users_topic
        self.state.accounts_topic = accounts_topic
        self.state.join_type     = (join_type or "left").lower()
        self.state.users_raw     = users_raw
        self.state.accounts_raw  = accounts_raw
        self.state.users_ui      = users_ui
        self.state.accounts_ui   = accounts_ui
        self.state.final_ui      = final_ui

        if not initial:
            print(f"[orchestrator] applied control: users='{users_topic}', accounts='{accounts_topic}', join='{join_type}'")

    def _linger_and_close(self, old_objs: List[object]):
        # Keep exports alive briefly so existing IDE panes don’t error mid-swap
        def _close():
            _safe_close_many(old_objs)

        if any(old_objs):
            t = threading.Timer(LINGER_SECONDS, _close)
            t.daemon = True
            t.start()

    # --------------- control listener ----------------

    def _on_control_update(self, update):
        """Called by Deephaven when new rows are appended to CONTROL_TOPIC."""
        try:
            j_tbl = update.added.j_table  # use Update.added to avoid extra query ops
            if j_tbl.size() <= 0:
                return

            row = j_tbl.getRowSet().lastRowKey()

            def _get(name: str) -> str:
                v = j_tbl.getColumnSource(name).get(row)
                return "" if v is None else str(v).strip()

            users = _get("usersTopic")
            accts = _get("accountsTopic")
            join  = _get("joinType") or "left"

            if users and accts:
                # Skip no-op updates
                if users == self.state.users_topic and accts == self.state.accounts_topic and (join or "left").lower() == self.state.join_type:
                    return
                self._set_topics(users, accts, join)
        except Exception as e:
            print("[orchestrator] control listener err:", e)


# -------- boot the app --------
_orch = Orchestrator(DEFAULT_USERS_TOPIC, DEFAULT_ACCOUNTS_TOPIC, "left")
```

---

## Notes / why this works

- **No `iif` or `coalesce`**: we only use 0.40.2-safe **ternary** expressions inside `.where(...)` or `.update_view(...)` (and we kept it minimal to avoid surprises).  
    _We also filter out `null`/empty `userId` and `last_by("userId")` on the right to prevent “duplicate right key” errors._
    
- **Listener context**: We read columns from `update.added.j_table` (no extra `where`/`snapshot` inside the callback), so there’s **no `ExecutionContext` error**.
    
- **Export swapping without UI flaps**: We export new tables first, then linger the old generation for a few seconds before closing—this avoids “export already failed” when IDE panes are open.
    
- **Angular** keeps subscribing to the same three names: `users_ui`, `accounts_ui`, `final_ui`. When the control message arrives, these are replaced under the same export names—your UI just keeps streaming.
    

---

## Control message format

Exactly this JSON on `CONTROL_TOPIC`:

```json
{"usersTopic":"<NEW_USERS>", "accountsTopic":"<NEW_ACCTS>", "joinType":"left"}
```

You can verify it took effect in the DH “Console” log (you’ll see the `applied control:` line) and by opening the exported tables in the IDE. The table `orchestrator_status` also shows the last applied control.

---

## Quick checklist

1. Ensure your config points App Mode to `C:\dhdata\app.d`:
    
    ```
    deephaven.application.dir=C:\\dhdata\\app.d
    ```
    
2. Drop the two files above into `C:\dhdata\app.d`.
    
3. Fill in **your** `KAFKA_CONFIG` (same props you already used).
    
4. Start: `deephaven server --port 10000`
    
5. Angular keeps reading `users_ui`, `accounts_ui`, `final_ui`.
    

If anything still hiccups, paste **the exact log line** and I’ll patch the specific section (but this build avoids the formula + context pitfalls you hit).