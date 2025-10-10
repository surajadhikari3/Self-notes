

Got it — the red “ExportObject … no longer live” is the app killing (or letting GC kill) objects that the IDE/Angular still has a handle to during the swap. The most robust fix is: **never tear down the previous generation immediately**. Keep strong references to the old generation for a while (and only retire them later), and always run swaps inside the app’s **ExecutionContext**. Below is a drop-in app for **Deephaven 0.40.2** that does exactly that.

It’s JSON-only, wraps all listener work in the correct context, de-dupes the right side for a left join, and **does not close** prior generations (so no liveness crashes). There’s an optional slow cleanup after a long delay if you want it.

---

# `C:\Deephaven\app.d\orchestrator.app`

```properties
id=kafka-orchestrator
name=Kafka Orchestrator
type=script
scriptType=python
enabled=true
file_0=orchestrator.py
```

# `C:\Deephaven\app.d\orchestrator.py`

```python
# Deephaven 0.40.2 – Kafka orchestrator (robust hot-swap, JSON-only control)
# - Hot-swaps topics via JSON control messages
# - Runs swaps inside the app's ExecutionContext
# - De-duplicates right side for safe left join
# - KEEPS previous generations (no immediate close) to avoid liveness errors
# - Optional slow retirement thread (disabled by default)
# - Exports: users_ui, accounts_ui, final_ui, control_raw, orchestrator_status

import math, threading, time
from contextlib import nullcontext
import pandas as pd

from deephaven.appmode import get_app_state, ApplicationState
from deephaven.pandas import to_pandas
from deephaven.table_listener import listen
from deephaven.stream.kafka import consumer as kc
import deephaven.dtypes as dt
from deephaven import agg as agg
from deephaven import new_table, time as dhtime
from deephaven.column import string_col
from deephaven import execution_context as ec

# ----- CONFIG (edit these to your env) ---------------------------------

DEFAULT_USERS_TOPIC    = "ccd01_sb_its_esp_tap3507_bishowcaseraw"
DEFAULT_ACCOUNTS_TOPIC = "ccd01_sb_its_esp_tap3507_bishowcasecurated"
CONTROL_TOPIC          = "ccd01_sb_its_esp_tap3507_metadata"

DATA_KAFKA = {
    "bootstrap.servers": "localhost:9092",
    "auto.offset.reset": "latest",
    # Security examples:
    # "security.protocol": "SASL_SSL",
    # "sasl.mechanism": "PLAIN",
    # "sasl.jaas.config": 'org.apache.kafka.common.security.plain.PlainLoginModule required username="<API_KEY>" password="<API_SECRET>";',
    # or OAuthBearer...
}

CONTROL_KAFKA = dict(DATA_KAFKA)
CONTROL_KAFKA["auto.offset.reset"] = "earliest"  # so we always see the last control row

# Keep old generations to prevent liveness errors
KEEP_GENERATIONS = True           # True = never close old gens automatically
RETIRE_AFTER_SECONDS = 900        # if you later set KEEP_GENERATIONS=False, close after this delay

# ----- END CONFIG ------------------------------------------------------

# Capture the app's ExecutionContext so callbacks have a QueryScope
_CTX = ec.get_exec_ctx()

USER_SPEC = kc.json_spec({
    "userId": dt.string,
    "name": dt.string,
    "email": dt.string,
    "age": dt.int64,
})

ACCOUNT_SPEC = kc.json_spec({
    "userId": dt.string,
    "accountType": dt.string,
    "balance": dt.double,
})

CONTROL_JSON_SPEC = kc.json_spec({
    "usersTopic": dt.string,
    "accountsTopic": dt.string,
    "joinType": dt.string,   # "left" (default) or "inner"
})

def consume_append(topic: str, spec, cfg):
    """Create an append-only streaming table from a Kafka topic."""
    return kc.consume(
        dict(cfg),
        topic,
        key_spec=kc.KeyValueSpec.IGNORE,
        value_spec=spec,
        table_type=kc.TableType.append(),
    )

def _scalar_str(val, default=""):
    """Robust string conversion (handles None, NaN, pd.NA)."""
    try:
        if val is None:
            return default
        try:
            if pd.isna(val):
                return default
        except Exception:
            pass
        if isinstance(val, float) and math.isnan(val):
            return default
        return str(val).strip()
    except Exception:
        return default

def _tbl_size(t):
    try:
        return t.size
    except Exception:
        try:
            return t.size()
        except Exception:
            return 0

class Orchestrator:
    def __init__(self, app: ApplicationState):
        self.app = app
        self.lock = threading.Lock()
        self.current_gen = None      # dict for the active generation
        self._last_topics = (None, None, None)  # users, accounts, join
        self._retired = []           # retired generations we keep around

    # --------- build a new generation of tables from topics ---------
    def _build_generation(self, users_topic: str, accounts_topic: str, join_type: str):
        gen = {"topics": (users_topic, accounts_topic, join_type), "resources": []}

        users_raw = consume_append(users_topic, USER_SPEC, DATA_KAFKA);     gen["resources"].append(users_raw)
        accts_raw = consume_append(accounts_topic, ACCOUNT_SPEC, DATA_KAFKA); gen["resources"].append(accts_raw)

        users = users_raw.view(["userId", "name", "email", "age"]);                     gen["resources"].append(users)
        accts = accts_raw.view(["userId", "accountType", "balance"]);                   gen["resources"].append(accts)

        # Ensure unique, non-null right keys for left join
        accts_non_null = accts.where("userId != null")
        try:
            accts_one = accts_non_null.last_by("userId")
        except AttributeError:
            accts_one = accts_non_null.agg_by([agg.last("accountType"), agg.last("balance")], by=["userId"])
        gen["resources"].append(accts_one)

        jt = (join_type or "left").lower()
        if jt.startswith("inner"):
            final = users.join(accts, on=["userId"], joins=["accountType", "balance"])
        else:
            final = users.natural_join(accts_one, on=["userId"], joins=["accountType", "balance"])
        gen["resources"].append(final)

        gen["users_ui"] = users
        gen["accounts_ui"] = accts
        gen["final_ui"] = final
        return gen

    # --------- swap to new topics (robust, no early close) ---------
    def set_topics(self, users_topic: str, accounts_topic: str, join_type: str = "left"):
        if not users_topic or not accounts_topic:
            raise ValueError("Both users_topic and accounts_topic are required")

        with self.lock, _CTX:
            # Build the new generation
            new_gen = self._build_generation(users_topic, accounts_topic, join_type)

            # Export under the SAME names (Angular/IDE keep working)
            self.app["users_ui"]    = new_gen["users_ui"]
            self.app["accounts_ui"] = new_gen["accounts_ui"]
            self.app["final_ui"]    = new_gen["final_ui"]

            self.app["orchestrator_status"] = new_table([
                string_col("usersTopic",   [users_topic]),
                string_col("accountsTopic",[accounts_topic]),
                string_col("joinType",     [(join_type or "left").lower()]),
                string_col("lastApplied",  [str(dhtime.dh_now())]),
            ])

            # Retire the old generation WITHOUT closing it (prevents liveness errors)
            if self.current_gen is not None:
                self._retire_generation(self.current_gen)

            self.current_gen = new_gen
            self._last_topics = (users_topic, accounts_topic, (join_type or "left").lower())
            print(f"[orchestrator] topics set users='{users_topic}' accounts='{accounts_topic}' join='{(join_type or 'left').lower()}'")

    def _retire_generation(self, gen):
        """Keep a strong reference so existing viewers don't crash."""
        self._retired.append((time.time(), gen))
        if KEEP_GENERATIONS:
            return
        # Optional slow retirement (only if KEEP_GENERATIONS==False)
        def _slow_close(stamp_gen):
            stamp, g = stamp_gen
            delay = max(1, RETIRE_AFTER_SECONDS)
            time.sleep(delay)
            for r in g.get("resources", []):
                try:
                    r.close()
                except Exception:
                    pass
            try:
                self._retired.remove(stamp_gen)
            except Exception:
                pass
        threading.Thread(target=_slow_close, args=((time.time(), gen),), daemon=True).start()

    # --------- control pipeline ---------
    def _parse_control_row(self, snap_df: pd.DataFrame):
        required = {"usersTopic", "accountsTopic"}
        if not required.issubset(set(snap_df.columns)):
            print("[orchestrator] control row missing columns:", list(snap_df.columns))
            return None
        row = snap_df.iloc[0]
        users = _scalar_str(row.get("usersTopic"))
        accts = _scalar_str(row.get("accountsTopic"))
        join  = _scalar_str(row.get("joinType"), "left") or "left"
        return (users, accts, join)

    def _apply_last_control(self, ctrl_tbl):
        snap = ctrl_tbl.tail(1).snapshot()
        if _tbl_size(snap) == 0:
            return
        df = to_pandas(snap)
        parsed = self._parse_control_row(df)
        if not parsed:
            return
        users, accts, join = parsed
        if users and accts and (users, accts, join) != self._last_topics:
            print(f"[orchestrator] applying control: users='{users}', accounts='{accts}', join='{join}'")
            self.set_topics(users, accts, join)

    def start_control_listener(self, control_topic: str):
        ctrl = consume_append(control_topic, CONTROL_JSON_SPEC, CONTROL_KAFKA)
        self.app["control_raw"] = ctrl  # visible in Apps panel

        def on_update(_update, _is_replay):
            try:
                with _CTX:
                    self._apply_last_control(ctrl)
            except Exception as e:
                print("[orchestrator] control listener err:", e)

        disp = listen(ctrl, on_update)  # (update, is_replay) signature

        # One initial apply (in case a row already exists)
        try:
            with _CTX:
                self._apply_last_control(ctrl)
        except Exception as e:
            print("[orchestrator] initial apply err:", e)

        # Keep references (not strictly needed, but harmless)
        # DO NOT close; we want robust behavior.
        self._retired.append(("control", {"resources": [ctrl, disp]}))
        print(f"[orchestrator] control listener on '{control_topic}'")

# ---------- boot ----------
_app = get_app_state()
_orc = Orchestrator(_app)

try:
    with _CTX:
        _orc.set_topics(DEFAULT_USERS_TOPIC, DEFAULT_ACCOUNTS_TOPIC, "left")
except Exception as boot_err:
    print("[orchestrator] initial wiring failed:", boot_err)

_orc.start_control_listener(CONTROL_TOPIC)
print("[orchestrator] ready")
```

---

## How to run

1. Save both files into `C:\Deephaven\app.d\`.
    
2. Start DH in your venv:
    
    ```bash
    source /c/Users/suraj/source/apps-testing/venv/Scripts/activate
    deephaven server --host localhost --port 10000
    ```
    
3. In the IDE → **Applications → Kafka Orchestrator**, open:
    
    - `users_ui`, `accounts_ui`, `final_ui`
        
    - `control_raw`
        
    - `orchestrator_status`
        

### Control message to publish (exact JSON)

```json
{"usersTopic":"<NEW_USERS>", "accountsTopic":"<NEW_ACCTS>", "joinType":"left"}
```

You should see:

```
[orchestrator] applying control: users='...' accounts='...' join='left'
[orchestrator] topics set users='...' accounts='...' join='left'
```

---

### Why this stops the crashes

- We **never close** the previous generation during the swap, so any open IDE/Angular viewers that still reference objects from the old generation remain valid (no `ExportObject... no longer live`).
    
- Exports keep the **same names** (`users_ui`, `accounts_ui`, `final_ui`), so Angular bindings keep working.
    
- All work triggered by the listener runs inside the app’s **ExecutionContext**, so `.where(...)`, joins, etc., are legal.
    
- Right-side de-duplication (`last_by("userId")` / `agg_by(last)`) prevents “duplicate right key” join failures.
    

If, later, you want automatic cleanup to reclaim memory, set `KEEP_GENERATIONS = False` and adjust `RETIRE_AFTER_SECONDS` (e.g., 600s). That retires generations **well after** viewers have had time to switch, without causing liveness faults.