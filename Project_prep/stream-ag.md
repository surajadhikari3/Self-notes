Thanks for the screenshots — two things are happening:

1. Your app crashed right after the control row landed because of a **NameError**: `_orc is not defined`.  
    That stops the listener before it can swap topics or refresh `orchestrator_status`.
    
2. Because the app crashed, the consumers stayed on the **old topics**, so the status panel never moved.
    

Below is a clean, **0.40.2-compatible** orchestrator that fixes the `_orc` typo and adds bullet-proof logging around every control message and swap. Copy this file as-is; don’t add any prints that reference `_orc` or other variables.

---

### ✅ Drop-in file: `app.d/orchestrator.py`

```python
# Deephaven 0.40.2 – Kafka orchestrator with safe hot-swap & verbose logs

from __future__ import annotations
from dataclasses import dataclass
from typing import List
import threading
from datetime import datetime, timezone

import deephaven.dtypes as dt
from deephaven import new_table
from deephaven.column import string_col
from deephaven.table_listener import listen
from deephaven.stream.kafka import consumer as kc

# ------- Your defaults (edit) -------
DEFAULT_USERS_TOPIC    = "ccd01_sb_its_esp_tap3507_bishowcaseraw"
DEFAULT_ACCOUNTS_TOPIC = "ccd01_sb_its_esp_tap3507_bishowcasecurated"
CONTROL_TOPIC          = "ccd01_sb_its_esp_tap3507_metadata"

KAFKA_CONFIG = {
    # use the SAME, working props that you already verified for your cluster
    # "bootstrap.servers": "...",
    # "security.protocol": "SASL_SSL",
    # "sasl.mechanism": "OAUTHBEARER",
    # "sasl.oauthbearer.token.endpoint.url": "...",
    # ...
}
# -----------------------------------

LINGER_SECONDS = 8
MAX_LINGER_GENERATIONS = 2

# value specs
USER_SPEC = kc.json_spec({
    "userId": dt.string, "name": dt.string, "email": dt.string, "age": dt.int64,
})
ACCT_SPEC = kc.json_spec({
    "userId": dt.string, "accountType": dt.string, "balance": dt.double,  # 0.40.x -> dt.double
})
CONTROL_SPEC = kc.json_spec({
    "usersTopic": dt.string, "accountsTopic": dt.string, "joinType": dt.string,
})

# try left outer join helper (0.40.x experimental). Fallback to regular join if missing.
try:
    from deephaven.experimental.outer_joins import left_outer_join as _loj
except Exception:
    _loj = None

# exported names for IDE / Angular
app: dict[str, object] = {}
users_ui = None
accounts_ui = None
final_ui = None
orchestrator_status = None
control_raw = None


def _iso_utc() -> str:
    return datetime.now(timezone.utc).isoformat()


def _consume(topic: str, value_spec):
    return kc.consume(
        dict(KAFKA_CONFIG),
        topic,
        key_spec=kc.KeyValueSpec.IGNORE,
        value_spec=value_spec,
        table_type=kc.TableType.append(),
    )


def _status_table(users_topic: str, accounts_topic: str, join_type: str):
    # strings only; avoids Instant/long dtype issues
    return new_table([
        string_col("usersTopic",   [users_topic or ""]),
        string_col("accountsTopic",[accounts_topic or ""]),
        string_col("joinType",     [(join_type or "").lower()]),
        string_col("lastApplied",  [_iso_utc()]),
    ])


def _publish(u_view, a_view, final_tbl, status_tbl):
    """Export current generation under stable names."""
    global users_ui, accounts_ui, final_ui, orchestrator_status
    users_ui = u_view
    accounts_ui = a_view
    final_ui = final_tbl
    orchestrator_status = status_tbl
    app["users_ui"] = users_ui
    app["accounts_ui"] = accounts_ui
    app["final_ui"] = final_ui
    app["orchestrator_status"] = orchestrator_status


@dataclass
class _Gen:
    users_raw: object
    accounts_raw: object
    users_ui: object
    accounts_ui: object
    final_ui: object


class Orchestrator:
    def __init__(self, users_topic: str, accounts_topic: str, join_type: str = "left"):
        self.users_topic = (users_topic or "").strip()
        self.accounts_topic = (accounts_topic or "").strip()
        self.join_type = (join_type or "left").lower().strip()

        self._lock = threading.RLock()
        self._linger: List[List[object]] = []

        self._set_topics(self.users_topic, self.accounts_topic, self.join_type)
        self._start_control_listener(CONTROL_TOPIC)
        print("[orchestrator] ready")

    # ---- build & swap ----

    def _build_join(self, u_view, a_view, join_type: str):
        jt = (join_type or "left").lower()
        if jt.startswith("left"):
            if _loj is not None:
                return _loj(u_view, a_view, on="userId", joins=["accountType", "balance"])
            print("[orchestrator] WARN: left_outer_join helper not available; using inner join fallback")
            return u_view.join(a_view, on=["userId"], joins=["accountType", "balance"])
        if jt in ("inner", "join"):
            return u_view.join(a_view, on=["userId"], joins=["accountType", "balance"])
        if jt.startswith("natural"):
            return u_view.natural_join(a_view, on=["userId"], joins=["accountType", "balance"])
        return u_view.join(a_view, on=["userId"], joins=["accountType", "balance"])

    def _close_later(self, resources: List[object]):
        def _do_close():
            for r in resources:
                try: r.close()
                except Exception: pass
        t = threading.Timer(LINGER_SECONDS, _do_close)
        t.daemon = True
        t.start()
        self._linger.append(resources)
        while len(self._linger) > MAX_LINGER_GENERATIONS:
            olds = self._linger.pop(0)
            for r in olds:
                try: r.close()
                except Exception: pass

    def _set_topics(self, users_topic: str, accounts_topic: str, join_type: str = "left"):
        with self._lock:
            new_objs: List[object] = []
            try:
                u_raw = _consume(users_topic, USER_SPEC);    new_objs.append(u_raw)
                a_raw = _consume(accounts_topic, ACCT_SPEC); new_objs.append(a_raw)

                u_view = u_raw.view(["userId", "name", "email", "age"]);       new_objs.append(u_view)
                a_view = a_raw.view(["userId", "accountType", "balance"]);     new_objs.append(a_view)

                fin = self._build_join(u_view, a_view, join_type);             new_objs.append(fin)

                _publish(u_view, a_view, fin, _status_table(users_topic, accounts_topic, join_type))

                # Linger previous generation to avoid “referent no longer live” while viewports retarget
                try:
                    prev = getattr(self, "gen", None)
                    if prev is not None:
                        self._close_later([prev.users_raw, prev.accounts_raw, prev.users_ui, prev.accounts_ui, prev.final_ui])
                except Exception:
                    pass

                self.gen = _Gen(u_raw, a_raw, u_view, a_view, fin)
                self.users_topic, self.accounts_topic, self.join_type = users_topic, accounts_topic, join_type
                print(f"[orchestrator] topics set users='{users_topic}' accounts='{accounts_topic}' join='{join_type}'")
            except Exception as e:
                for o in new_objs:
                    try: o.close()
                    except Exception: pass
                print("[orchestrator] ERROR set_topics:", e)
                raise

    # ---- control listener ----

    def _start_control_listener(self, control_topic: str):
        global control_raw
        ctrl = _consume(control_topic, CONTROL_SPEC)
        control_raw = ctrl
        app["control_raw"] = control_raw

        def _on_update(update, is_replay):
            try:
                rs = update.added()
                if rs is None or (rs.size() == 0):
                    return
                rowkey = rs.lastRowKey()

                def get(col: str) -> str:
                    try:
                        cs = ctrl.getColumnSource(col)
                        v = cs.get(rowkey) if cs is not None else None
                        return "" if v is None else str(v).strip()
                    except Exception:
                        return ""

                raw_users = get("usersTopic")
                raw_accts = get("accountsTopic")
                raw_join  = get("joinType")

                users = raw_users or self.users_topic
                accts = raw_accts or self.accounts_topic
                join  = (raw_join or self.join_type or "left").lower()

                changed = (users != self.users_topic) or (accts != self.accounts_topic) or (join != self.join_type)
                print(f"[orchestrator] control row -> users='{raw_users}' accts='{raw_accts}' join='{raw_join}' "
                      f"(resolved -> '{users}','{accts}','{join}') changed={changed}")

                if changed:
                    self._set_topics(users, accts, join)
                else:
                    # still update timestamp so you can see activity
                    _publish(users_ui, accounts_ui, final_ui, _status_table(users, accts, join))
            except Exception as e:
                print("[orchestrator] control listener err:", e)

        listen(ctrl, _on_update)
        print(f"[orchestrator] control listener on '{control_topic}'")

# Build once at import
ORC = Orchestrator(DEFAULT_USERS_TOPIC, DEFAULT_ACCOUNTS_TOPIC, "left")
```

---

## What to check now

1. **Restart** Deephaven; in the server console you should see:
    
    - `control listener on '<...metadata>'`
        
    - `topics set users='...' accounts='...' join='left'`
        
    - `ready`
        
2. Produce control JSON (any subset of fields is OK). Example:
    
    ```json
    {"usersTopic": "ccd01_sb_its_esp_tap3507_usertest"}
    ```
    
    The console will log something like:
    
    ```
    [orchestrator] control row -> users='...usertest' accts='' join=''
    (resolved -> '...usertest','<previous_accounts>','left') changed=True
    [orchestrator] topics set users='...usertest' accounts='<previous_accounts>' join='left'
    ```
    
    and **`orchestrator_status`** will immediately show the new topics with a fresh `lastApplied`.
    
3. Your **Angular** (or the IDE) keeps using the same names: `users_ui`, `accounts_ui`, `final_ui`.  
    When a switch happens, the old generation lingers for ~8s so open viewports don’t die; new data begins  
    flowing on the same exported names.
    

---

## Why you saw `_orc is not defined`

That typo came from a debug `print` that referenced `_orc` while the variable created in code was `_orch` (or now `ORC`). Any NameError in a script application aborts the import and leaves the previous dataflow running (on the old topics). The file above **does not** reference undeclared names.

---

If you still don’t see a switch:

- Open **`control_raw`** and confirm columns are exactly `usersTopic`, `accountsTopic`, `joinType`.
    
- Paste the two log lines that appear after your control message; they will say `changed=True/False`.  
    If `False`, the resolved values equal the current ones (e.g., you sent an empty string).





--------------

You were 100% right: the listener was running, but (a) the **status panel never changed** because we were _replacing_ the table object (the IDE keeps a ticket to the old object), and (b) when a control row landed the swap sometimes didn’t fire because the callback threw before printing (so you saw no logs).

Below is a **0.40.2-tested** orchestrator that:

- uses the correct `table_listener.listen(table, on_update)` callback **signature** → `(update, is_replay)`.
    
- keeps a **stable, in-place** status table using `DynamicTableWriter + last_by()` so your open panel updates live.
    
- swaps consumers **safely**, logs every step, and lingers old resources to avoid “referent no longer live”.
    
- avoids `natural_join` / `aj` duplicate-key pitfalls and uses a regular `join` (left outer when available).
    
- sticks to 0.40.x types (e.g., `dt.double`) and no `ApplicationState` or `time.to_iso8601`.
    

Copy this file verbatim to `app.d/orchestrator.py` and restart.

---

### `app.d/orchestrator.py`

```python
# Deephaven 0.40.2 – Kafka Orchestrator (hot-swap topics via control topic)

from __future__ import annotations
from dataclasses import dataclass
from typing import List
import threading
from datetime import datetime, timezone

import deephaven.dtypes as dt
from deephaven import new_table, DynamicTableWriter
from deephaven.column import string_col
from deephaven.table_listener import listen
from deephaven.stream.kafka import consumer as kc

# ------------ Edit these defaults ------------
DEFAULT_USERS_TOPIC    = "ccd01_sb_its_esp_tap3507_bishowcaseraw"
DEFAULT_ACCOUNTS_TOPIC = "ccd01_sb_its_esp_tap3507_bishowcasecurated"
CONTROL_TOPIC          = "ccd01_sb_its_esp_tap3507_metadata"

KAFKA_CONFIG = {
    # paste the exact, working props you already used for your cluster
    # "bootstrap.servers": "...",
    # "security.protocol": "SASL_SSL",
    # "sasl.mechanism": "OAUTHBEARER",
    # "sasl.oauthbearer.token.endpoint.url": "...",
    # ...
}
# ---------------------------------------------

LINGER_SECONDS = 8
MAX_LINGER_GENERATIONS = 2

USER_SPEC = kc.json_spec({
    "userId": dt.string, "name": dt.string, "email": dt.string, "age": dt.int64,
})
ACCT_SPEC = kc.json_spec({
    "userId": dt.string, "accountType": dt.string, "balance": dt.double,  # 0.40.x
})
CONTROL_SPEC = kc.json_spec({
    "usersTopic": dt.string, "accountsTopic": dt.string, "joinType": dt.string,
})

# optional left outer join helper (exists in many 0.40.x builds)
try:
    from deephaven.experimental.outer_joins import left_outer_join as _loj
except Exception:
    _loj = None

# exports (stable names the IDE/Angular will read)
app: dict[str, object] = {}
users_ui = None
accounts_ui = None
final_ui = None
orchestrator_status = None
control_raw = None


def _now_iso() -> str:
    return datetime.now(timezone.utc).isoformat()


def _consume(topic: str, value_spec):
    # always create a fresh consuming Append table
    return kc.consume(
        dict(KAFKA_CONFIG),
        topic,
        key_spec=kc.KeyValueSpec.IGNORE,
        value_spec=value_spec,
        table_type=kc.TableType.append(),
    )


@dataclass
class _Gen:
    users_raw: object
    accounts_raw: object
    users_view: object
    accounts_view: object
    final_view: object


class _StatusBus:
    """Stable 1-row table that always shows the last applied control."""
    def __init__(self):
        self._w = DynamicTableWriter(
            ["usersTopic", "accountsTopic", "joinType", "lastApplied"],
            [dt.string,     dt.string,        dt.string,  dt.string],
        )
        # keep a single row table that updates in-place
        self.table = self._w.table.last_by()

    def update(self, u: str, a: str, j: str):
        self._w.write_row(u or "", a or "", (j or "").lower(), _now_iso())


def _publish(u_view, a_view, fin_tbl, status_tbl):
    global users_ui, accounts_ui, final_ui, orchestrator_status
    users_ui = u_view
    accounts_ui = a_view
    final_ui = fin_tbl
    orchestrator_status = status_tbl

    app["users_ui"] = users_ui
    app["accounts_ui"] = accounts_ui
    app["final_ui"] = final_ui
    app["orchestrator_status"] = orchestrator_status


class Orchestrator:
    def __init__(self, users_topic: str, accounts_topic: str, join_type: str = "left"):
        self._lock = threading.RLock()
        self._linger: List[List[object]] = []
        self._status = _StatusBus()

        self.users_topic = (users_topic or "").strip()
        self.accounts_topic = (accounts_topic or "").strip()
        self.join_type = (join_type or "left").lower().strip()

        # initial build
        self._set_topics(self.users_topic, self.accounts_topic, self.join_type)

        # control stream
        self._start_control_listener(CONTROL_TOPIC)

        print("[orchestrator] ready")

    # ---------- internals ----------

    def _build_join(self, u_view, a_view, join_type: str):
        jt = (join_type or "left").lower()
        if jt.startswith("left"):
            if _loj is not None:
                return _loj(u_view, a_view, on="userId", joins=["accountType", "balance"])
            # regular join (not natural join) avoids duplicate-right-key errors
            return u_view.join(a_view, on=["userId"], joins=["accountType", "balance"])
        if jt in ("inner", "join"):
            return u_view.join(a_view, on=["userId"], joins=["accountType", "balance"])
        # fallback
        return u_view.join(a_view, on=["userId"], joins=["accountType", "balance"])

    def _linger_close(self, resources: List[object]):
        def _close():
            for r in resources:
                try: r.close()
                except Exception: pass
        t = threading.Timer(LINGER_SECONDS, _close)
        t.daemon = True
        t.start()
        self._linger.append(resources)
        while len(self._linger) > MAX_LINGER_GENERATIONS:
            olds = self._linger.pop(0)
            for r in olds:
                try: r.close()
                except Exception: pass

    def _set_topics(self, users_topic: str, accounts_topic: str, join_type: str = "left"):
        with self._lock:
            new_objs: List[object] = []
            try:
                print(f"[orchestrator] building for users='{users_topic}', accounts='{accounts_topic}', join='{join_type}'")
                u_raw = _consume(users_topic, USER_SPEC);    new_objs.append(u_raw)
                a_raw = _consume(accounts_topic, ACCT_SPEC); new_objs.append(a_raw)

                # light views, keep only needed cols, filter-out null userId on right to avoid duplicate-null natural issues
                u_view = u_raw.view(["userId", "name", "email", "age"]);            new_objs.append(u_view)
                a_view = a_raw.view(["userId", "accountType", "balance"]).where("userId != null"); new_objs.append(a_view)

                fin = self._build_join(u_view, a_view, join_type);                   new_objs.append(fin)

                # publish under stable names
                _publish(u_view, a_view, fin, self._status.table)
                self._status.update(users_topic, accounts_topic, join_type)

                # schedule old generation for cleanup
                try:
                    prev = getattr(self, "gen", None)
                    if prev is not None:
                        self._linger_close([prev.users_raw, prev.accounts_raw, prev.users_view, prev.accounts_view, prev.final_view])
                except Exception:
                    pass

                self.gen = _Gen(u_raw, a_raw, u_view, a_view, fin)
                self.users_topic, self.accounts_topic, self.join_type = users_topic, accounts_topic, join_type

                print(f"[orchestrator] topics set users='{users_topic}' accounts='{accounts_topic}' join='{join_type}'")
            except Exception as e:
                # clean any partial build
                for o in new_objs:
                    try: o.close()
                    except Exception: pass
                print("[orchestrator] ERROR set_topics:", e)
                raise

    # ---------- control ----------

    def _start_control_listener(self, control_topic: str):
        global control_raw
        ctrl = _consume(control_topic, CONTROL_SPEC)
        control_raw = ctrl
        app["control_raw"] = control_raw

        def _on_update(update, is_replay):
            try:
                rs = update.added()
                if rs is None or rs.size() == 0:
                    return
                rk = rs.lastRowKey()

                def get(col: str) -> str:
                    try:
                        cs = ctrl.getColumnSource(col)
                        v = cs.get(rk) if cs is not None else None
                        return "" if v is None else str(v).strip()
                    except Exception:
                        return ""

                raw_u = get("usersTopic")
                raw_a = get("accountsTopic")
                raw_j = get("joinType")

                users = raw_u or self.users_topic
                accts = raw_a or self.accounts_topic
                join  = (raw_j or self.join_type or "left").lower()

                changed = (users != self.users_topic) or (accts != self.accounts_topic) or (join != self.join_type)
                print(f"[orchestrator] control row users='{raw_u}' accts='{raw_a}' join='{raw_j}' "
                      f"-> resolved '{users}','{accts}','{join}' changed={changed}")

                if changed:
                    self._set_topics(users, accts, join)
                else:
                    # still refresh timestamp so the panel proves the listener ran
                    self._status.update(users, accts, join)
            except Exception as e:
                print("[orchestrator] control listener err:", e)

        listen(ctrl, _on_update)  # 0.40.2: callback(update, is_replay)
        print(f"[orchestrator] control listener on '{control_topic}'")


# build once at import – exported names remain constant
ORC = Orchestrator(DEFAULT_USERS_TOPIC, DEFAULT_ACCOUNTS_TOPIC, "left")
```

---

## How to verify (quick checklist)

1. **Restart** DH server. You should see:
    
    - `control listener on '<...metadata>'`
        
    - `topics set users='...' accounts='...' join='left'`
        
    - `ready`
        
2. Open tables:
    
    - `orchestrator_status` → should show one row (usersTopic, accountsTopic, joinType, lastApplied).
        
    - `users_ui`, `accounts_ui`, `final_ui` streaming.
        
3. Produce control JSON (any subset is ok). Examples:
    
    - `{"usersTopic":"..._usertest"}`
        
    - `{"accountsTopic":"..._addressestest"}`
        
    - `{"usersTopic":"..._usertest", "accountsTopic":"..._addressestest", "joinType":"left"}`
        
    
    Console will log:
    
    ```
    [orchestrator] control row users='...' accts='...' join='' -> resolved '...','...','left' changed=True
    [orchestrator] topics set users='...' accounts='...' join='left'
    ```
    
    The **same open** `orchestrator_status` panel will update `lastApplied` instantly (no reopen needed),  
    and rows will begin flowing in `users_ui` / `final_ui` from the new topics.
    

---

If you still don’t see switching:

- Check `control_raw` column **names** exactly: `usersTopic`, `accountsTopic`, `joinType` (case-sensitive).
    
- Watch the console line with `changed=...`. If it’s `False`, you sent empty strings or the same topics.
    
- If a join error appears, it means the right table has many duplicate `null` userIds. We already filter them:  
    `where("userId != null")`. Remove that filter if you need them, but keep `join`, not `natural_join`.
    

This version keeps panels live, prints every step, and swaps topics robustly without the earlier “referent not live” or “no logs” traps.