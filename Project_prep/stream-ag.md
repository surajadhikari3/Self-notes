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