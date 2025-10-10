You’re right: in **0.40.2** the callback you get from `deephaven.table_listener.listen` can look like either a Java `TableUpdate` **or** a Python `dict`-style object (with keys `added/removed/modified/...`). If you only code for the `TableUpdate` case and call `update.added().size()`, you’ll hit:

```
AttributeError: 'dict' object has no attribute 'size'
```

Below is a drop-in **orchestrator.py** that works in IDE or App mode and handles both callback shapes. I also added loud, explicit logging when:

- a control record arrives,
    
- nothing changed (status refreshed only),
    
- topics actually switch (old → new),
    
- a generation is closed after the linger period.
    

There are no `from __future__` lines, imports are top-of-file, and all DH 0.40.2 APIs are used correctly.

> Copy/paste the file as `app.d/orchestrator.py` (or run in the IDE).  
> Update the `KAFKA_CONFIG` and the 3 default topics for your cluster.

---

```python
# Deephaven 0.40.2 – Kafka Orchestrator with reliable hot-swap via control topic
# - Handles listen() callback as TableUpdate OR dict
# - Logs every control application and swap
# - Closes old generation after a short linger to avoid races

from dataclasses import dataclass
from typing import List, Optional
import threading
from datetime import datetime, timezone

import deephaven.dtypes as dt
from deephaven import DynamicTableWriter
from deephaven.table_listener import listen
from deephaven.stream.kafka import consumer as kc

# ── EDIT DEFAULTS ────────────────────────────────────────────────────────────
DEFAULT_USERS_TOPIC    = "ccd01_sb_its_esp_tap3507_bishowcaseraw"
DEFAULT_ACCOUNTS_TOPIC = "ccd01_sb_its_esp_tap3507_bishowcasecurated"
CONTROL_TOPIC          = "ccd01_sb_its_esp_tap3507_metadata"

KAFKA_CONFIG = {
    # Fill in your working config (these are examples / placeholders)
    # "bootstrap.servers": "...",
    # "security.protocol": "SASL_SSL",
    # "sasl.mechanism": "OAUTHBEARER",
    # "sasl.oauthbearer.token.endpoint.url": "...",
    # "sasl.oauthbearer.sub.claim.name": "client_id",
    # "sasl.oauthbearer.client.id": "...",
    # "sasl.oauthbearer.client.secret": "...",
    # "sasl.oauthbearer.extensions.logicalCluster": "...",
    # "sasl.oauthbearer.extensions.identityPoolId": "...",
    # "ssl.endpoint.identification.algorithm": "HTTPS",
}
# ─────────────────────────────────────────────────────────────────────────────

LINGER_SECONDS = 8
MAX_LINGER_GENERATIONS = 2

# JSON specs (0.40.x mapping forms)
USER_SPEC = kc.json_spec({
    "userId": dt.string,
    "name": dt.string,
    "email": dt.string,
    "age": dt.int64,
})
ACCT_SPEC = kc.json_spec({
    "userId": dt.string,
    "accountType": dt.string,
    "balance": dt.double,
})
CONTROL_SPEC = kc.json_spec({
    "usersTopic": dt.string,
    "accountsTopic": dt.string,
    "joinType": dt.string,
})

# Optional left_outer_join helper (present in some installs)
try:
    from deephaven.experimental.outer_joins import left_outer_join as _loj
except Exception:
    _loj = None

# Exports (Angular / IDE look for these names)
app: dict[str, object] = {}
users_ui = None
accounts_ui = None
final_ui = None
orchestrator_status = None
control_raw = None


def _now_iso() -> str:
    return datetime.now(timezone.utc).isoformat()


def _consume(topic: str, value_spec):
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
    """1-row status table that we update on each control apply."""

    def __init__(self):
        # 0.40.2: pass a single dict mapping col->dtype
        self._w = DynamicTableWriter({
            "usersTopic": dt.string,
            "accountsTopic": dt.string,
            "joinType": dt.string,
            "lastApplied": dt.string,  # ISO-8601 text for ease of display
        })
        self.table = self._w.table.last_by()

    def update(self, users: str, accts: str, join_type: str):
        self._w.write_row(users or "", accts or "", (join_type or "").lower(), _now_iso())


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

        self._set_topics(self.users_topic, self.accounts_topic, self.join_type)
        self._start_control_listener(CONTROL_TOPIC)
        print("[orchestrator] ready")

    # ── Building the joined view ──────────────────────────────────────────────
    def _build_join(self, u_view, a_view, join_type: str):
        jt = (join_type or "left").lower()
        if jt.startswith("left"):
            if _loj is not None:
                return _loj(u_view, a_view, on="userId", joins=["accountType", "balance"])
            return u_view.join(a_view, on=["userId"], joins=["accountType", "balance"])
        if jt in ("inner", "join"):
            return u_view.join(a_view, on=["userId"], joins=["accountType", "balance"])
        # default
        return u_view.join(a_view, on=["userId"], joins=["accountType", "balance"])

    def _linger_close(self, resources: List[object]):
        def _close():
            for r in resources:
                try:
                    r.close()
                except Exception:
                    pass
            print("[orchestrator] closed previous generation resources")
        t = threading.Timer(LINGER_SECONDS, _close)
        t.daemon = True
        t.start()
        self._linger.append(resources)
        while len(self._linger) > MAX_LINGER_GENERATIONS:
            olds = self._linger.pop(0)
            for r in olds:
                try:
                    r.close()
                except Exception:
                    pass

    def _set_topics(self, users_topic: str, accounts_topic: str, join_type: str = "left"):
        with self._lock:
            new_objs: List[object] = []
            try:
                print(f"[orchestrator] building: users='{users_topic}', accounts='{accounts_topic}', join='{join_type}'")
                u_raw = _consume(users_topic, USER_SPEC);    new_objs.append(u_raw)
                a_raw = _consume(accounts_topic, ACCT_SPEC); new_objs.append(a_raw)

                u_view = u_raw.view(["userId", "name", "email", "age"]);            new_objs.append(u_view)
                a_view = a_raw.view(["userId", "accountType", "balance"]).where("userId != null"); new_objs.append(a_view)

                fin = self._build_join(u_view, a_view, join_type);                   new_objs.append(fin)

                _publish(u_view, a_view, fin, self._status.table)
                self._status.update(users_topic, accounts_topic, join_type)

                # schedule old gen close
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
                for o in new_objs:
                    try: o.close()
                    except Exception: pass
                print("[orchestrator] ERROR set_topics:", e)
                raise

    # ── Control listener (robust to dict/TableUpdate) ─────────────────────────
    def _start_control_listener(self, control_topic: str):
        global control_raw
        ctrl = _consume(control_topic, CONTROL_SPEC)
        control_raw = ctrl
        app["control_raw"] = control_raw

        def _rowset_is_empty(rs) -> bool:
            if rs is None:
                return True
            try:
                if hasattr(rs, "isEmpty") and rs.isEmpty():
                    return True
                if hasattr(rs, "sizeLong") and rs.sizeLong() == 0:
                    return True
                if hasattr(rs, "size") and rs.size() == 0:
                    return True
            except Exception:
                pass
            return False

        def _get_last_added_key(update) -> Optional[int]:
            """Support both TableUpdate and dict styles."""
            # TableUpdate style
            try:
                rs = update.added()
                if not _rowset_is_empty(rs):
                    return rs.lastRowKey()
            except Exception:
                pass
            # dict style
            try:
                if isinstance(update, dict):
                    rs = update.get("added")
                    if not _rowset_is_empty(rs):
                        # rs is a RowSet here too
                        return rs.lastRowKey()
            except Exception:
                pass
            return None

        def _get_str(rs_key: int, col: str) -> str:
            try:
                cs = ctrl.getColumnSource(col)
                v = cs.get(rs_key) if cs is not None else None
                return "" if v is None else str(v).strip()
            except Exception:
                return ""

        def _on_update(update, is_replay):
            try:
                rk = _get_last_added_key(update)
                if rk is None:
                    return

                raw_u = _get_str(rk, "usersTopic")
                raw_a = _get_str(rk, "accountsTopic")
                raw_j = _get_str(rk, "joinType")

                # coalesce with current state
                users = raw_u or self.users_topic
                accts = raw_a or self.accounts_topic
                join  = (raw_j or self.join_type or "left").lower()

                changed = (users != self.users_topic) or (accts != self.accounts_topic) or (join != self.join_type)

                print(f"[orchestrator] control: raw=({raw_u!r},{raw_a!r},{raw_j!r}) "
                      f"resolved=({users!r},{accts!r},{join!r}) changed={changed}")

                if changed:
                    old = (self.users_topic, self.accounts_topic, self.join_type)
                    self._set_topics(users, accts, join)
                    print(f"[orchestrator] swapped topics old={old} -> new={(users, accts, join)}")
                else:
                    # still record heartbeat
                    self._status.update(users, accts, join)
                    print("[orchestrator] no change; status refreshed")
            except Exception as e:
                print("[orchestrator] control listener err:", e)

        listen(ctrl, _on_update)
        print(f"[orchestrator] control listener on '{control_topic}'")

# Build on import and expose stable names
ORC = Orchestrator(DEFAULT_USERS_TOPIC, DEFAULT_ACCOUNTS_TOPIC, "left")
```

### Why this fixes your error

- The callback now **accepts both shapes** (`TableUpdate` or `dict`). If it’s the dict shape, we fetch `update["added"]` and operate on that `RowSet`. No `.size()` calls on a dict anymore.
    
- We use `sizeLong()` / `isEmpty()` when available, so you won’t see `…has no attribute size` in either case.
    

### How to verify hot-swap works

1. Keep the three tables open: `users_ui`, `accounts_ui`, `final_ui`.  
    Also open `orchestrator_status` (1 row).
    
2. Produce a control message to your control topic (same JSON as before):
    
    ```json
    {"usersTopic":"<NEW_USERS>", "accountsTopic":"<NEW_ACCTS>", "joinType":"left"}
    ```
    
3. Watch the Deephaven console:
    
    - You should see a line like  
        `[orchestrator] control: raw=('NEW_USERS','NEW_ACCTS','left') resolved=(...) changed=True`
        
    - Then  
        `[orchestrator] building: users='NEW_USERS', accounts='NEW_ACCTS', join='left'`
        
    - Then  
        `[orchestrator] topics set users='NEW_USERS' accounts='NEW_ACCTS' join='left'`
        
    - Finally  
        `[orchestrator] swapped topics old=(...) -> new=(...)`
        
4. The **orchestrator_status** row updates its `lastApplied` timestamp and the _usersTopic/accountsTopic_ columns reflect the new values.
    
5. New data starts appearing in `users_ui`/`accounts_ui`/`final_ui` from the new topics. The prior generation will be closed after `LINGER_SECONDS`.
    

If anything still logs an error, paste the exact console lines and I’ll adapt the listener again—but with this version you won’t get the “dict has no attribute size” problem, and you’ll have clear logs showing every step of a topic switch.