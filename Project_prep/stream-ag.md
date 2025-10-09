Absolutely—here’s a **ready-to-paste** App-Mode package for **Deephaven 0.40.2** that:

- Listens to two Kafka topics and builds `users_ui`, `accounts_ui`, and a **safe left join** `final_ui`
    
- Hot-swaps streams when you publish a **JSON** control message
    
- Avoids “liveness / dependency cancelled” errors by doing a **graceful hand-off**
    
- Exposes `control_raw` so you can see control messages arrive
    
- Keeps the **same export names** so your Angular client (using the IDE session) keeps working
    

---

# 1) `C:\Deephaven\app.d\orchestrator.app`

```properties
id=kafka-orchestrator
name=Kafka Orchestrator
type=script
scriptType=python
enabled=true
file_0=orchestrator.py
```

# 2) `C:\Deephaven\app.d\orchestrator.py`

```python
# Deephaven 0.40.2 – Kafka orchestrator with JSON-only control topic
# - Hot-swaps topics via control JSON
# - Graceful hand-off to avoid UI/Angular failures
# - Left join with de-duplicated right table

import math, threading, time
import pandas as pd

from deephaven.appmode import get_app_state, ApplicationState
from deephaven.pandas import to_pandas
from deephaven.table_listener import listen
from deephaven.stream.kafka import consumer as kc
import deephaven.dtypes as dt
from deephaven import agg as agg
from deephaven import new_table, time as dhtime
from deephaven.column import string_col

# -------- EDIT THESE (your real topics + Kafka props) -----------------

DEFAULT_USERS_TOPIC    = "ccd01_sb_its_esp_tap3507_bishowcaseraw"
DEFAULT_ACCOUNTS_TOPIC = "ccd01_sb_its_esp_tap3507_bishowcasecurated"
CONTROL_TOPIC          = "ccd01_sb_its_esp_tap3507_metadata"

# Data consumers (usually "latest")
DATA_KAFKA = {
    "bootstrap.servers": "localhost:9092",
    "auto.offset.reset": "latest",
    # SECURITY EXAMPLES:
    # Confluent Cloud (PLAIN):
    # "security.protocol": "SASL_SSL",
    # "sasl.mechanism": "PLAIN",
    # "sasl.jaas.config": 'org.apache.kafka.common.security.plain.PlainLoginModule required username="<API_KEY>" password="<API_SECRET>";',
    #
    # OIDC / OAuthBearer:
    # "security.protocol": "SASL_SSL",
    # "sasl.mechanism": "OAUTHBEARER",
    # "sasl.login.callback.handler.class": "io.confluent.kafka.clients.plugins.auth.token.TokenUserLoginCallbackHandler",
    # ... your token props ...
}

# Control consumer: ALWAYS start at earliest so we can apply the last-known row
CONTROL_KAFKA = dict(DATA_KAFKA)
CONTROL_KAFKA["auto.offset.reset"] = "earliest"

# ---------------------------------------------------------------------

# Value specs
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


class Orchestrator:
    def __init__(self, app: ApplicationState):
        self.app = app
        self.lock = threading.Lock()
        self.resources = []            # current generation (tables, listeners)
        self._last_topics = (None, None, None)  # users, accounts, join

    # ---------- core hot-swap ----------
    def set_topics(self, users_topic: str, accounts_topic: str, join_type: str = "left"):
        if not users_topic or not accounts_topic:
            raise ValueError("Both users_topic and accounts_topic are required")

        with self.lock:
            old_resources = list(self.resources)  # close after a short delay

            new_res = []
            try:
                # raw streams
                users_raw  = consume_append(users_topic,  USER_SPEC,    DATA_KAFKA); new_res.append(users_raw)
                accts_raw  = consume_append(accounts_topic, ACCOUNT_SPEC, DATA_KAFKA); new_res.append(accts_raw)

                # projections
                users = users_raw.view(["userId", "name", "email", "age"]);                  new_res.append(users)
                accts = accts_raw.view(["userId", "accountType", "balance"]);               new_res.append(accts)

                # right side must be unique & non-null on userId for natural_join
                accts_non_null = accts.where("userId != null")
                try:
                    accts_one = accts_non_null.last_by("userId")
                except AttributeError:
                    accts_one = accts_non_null.agg_by([agg.last("accountType"), agg.last("balance")], by=["userId"])
                new_res.append(accts_one)

                jt = (join_type or "left").lower()
                if jt.startswith("inner"):
                    final = users.join(accts, on=["userId"], joins=["accountType", "balance"])
                else:
                    final = users.natural_join(accts_one, on=["userId"], joins=["accountType", "balance"])
                new_res.append(final)

                # export new generation (global names remain constant)
                self.app["users_ui"]    = users
                self.app["accounts_ui"] = accts
                self.app["final_ui"]    = final

                # tiny status table (handy for Angular / quick checks)
                self.app["orchestrator_status"] = new_table([
                    string_col("usersTopic",   [users_topic]),
                    string_col("accountsTopic",[accounts_topic]),
                    string_col("joinType",     [jt]),
                    string_col("lastApplied",  [str(dhtime.dh_now())]),
                ])

                # publish as current generation
                self.resources = new_res
                self._last_topics = (users_topic, accounts_topic, jt)
                print(f"[orchestrator] topics set users='{users_topic}' accounts='{accounts_topic}' join='{jt}'")

                # graceful hand-off: close old gen after a brief delay so UI/Angular can rebind
                def _close_after_delay(res_list):
                    try:
                        time.sleep(2.0)  # 1–3s is typical
                        for r in res_list:
                            try:
                                r.close()
                            except Exception:
                                pass
                    except Exception:
                        pass

                threading.Thread(target=_close_after_delay, args=(old_resources,), daemon=True).start()

            except Exception as e:
                # Roll back anything new we created
                for r in new_res:
                    try:
                        r.close()
                    except Exception:
                        pass
                print("[orchestrator] set_topics error:", e)
                raise

    # ---------- control pipeline ----------
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
        if snap.size == 0:
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
                self._apply_last_control(ctrl)
            except Exception as e:
                print("[orchestrator] control listener err:", e)

        disp = listen(ctrl, on_update)

        # one initial apply (if a row already exists)
        try:
            self._apply_last_control(ctrl)
        except Exception as e:
            print("[orchestrator] initial apply err:", e)

        self.resources.extend([ctrl, disp])
        print(f"[orchestrator] control listener on '{control_topic}'")

# ---------- boot ----------
_app = get_app_state()
_orc = Orchestrator(_app)
try:
    _orc.set_topics(DEFAULT_USERS_TOPIC, DEFAULT_ACCOUNTS_TOPIC, "left")
except Exception as boot_err:
    print("[orchestrator] initial wiring failed:", boot_err)
_orc.start_control_listener(CONTROL_TOPIC)
print("[orchestrator] ready")
```

---

## 3) Start & test

```bash
# activate your venv
source /c/Users/suraj/source/apps-testing/venv/Scripts/activate
deephaven server --host localhost --port 10000
```

Open **[http://localhost:10000/ide/](http://localhost:10000/ide/)** → Panels → **Applications → Kafka Orchestrator** → open:

- `control_raw`
    
- `users_ui`, `accounts_ui`, `final_ui`
    
- (optional) `orchestrator_status`
    

### Control message to publish (exact JSON)

```json
{"usersTopic":"<NEW_USERS>", "accountsTopic":"<NEW_ACCTS>", "joinType":"left"}
```

- Keys are **case-sensitive**.
    
- `joinType` may be `"left"` (default) or `"inner"`.
    

**Expected logs** when a control row arrives:

```
[orchestrator] applying control: users='<NEW_USERS>' accounts='<NEW_ACCTS>' join='left'
[orchestrator] topics set users='<NEW_USERS>' accounts='<NEW_ACCTS>' join='left'
```

The exports keep their names, so your **Angular client** (reading via the IDE session) continues to work with the updated streams.

---

### Notes / common pitfalls

- If `control_raw` shows **no rows**, the consumer didn’t get your message (topic, ACLs, or security settings).
    
- If `final_ui` ever fails again, the accounts stream likely had duplicate or null `userId`. We already:
    
    - `where("userId != null")`
        
    - `last_by("userId")` (or `agg_by(last)` fallback)
        
- If you emit many control messages back-to-back, the 2s grace period handles UI/Angular rebinding cleanly. Tune to 1–3s if needed.
    

This version removes the `simple_spec` path entirely (JSON-only), matches 0.40.2 APIs, and includes the graceful hand-off to eliminate the “liveness / dependency cancelled” noise.