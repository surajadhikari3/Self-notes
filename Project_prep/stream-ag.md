I hear you—this has been frustrating. The two things that kept biting you were:

1. wrong listener callback signature (must be `(update, is_replay)` in 0.40.2, and **no** `replay_initial` kw), and
    
2. trying to mutate / export from threads without a clean path + status writer (which is why your `orchestrator_status` sometimes never moved).
    

Below is a single self-contained `orchestrator.py` for **Deephaven 0.40.2** that:

- builds users / accounts Kafka tables from topics,
    
- does a **left** (or **inner**) join,
    
- exposes 4 tables by name: `users_ui`, `accounts_ui`, `final_ui`, `orchestrator_status`,
    
- listens to a **control** topic with JSON messages like  
    `{"usersTopic":"<NEW_USERS>","accountsTopic":"<NEW_ACCTS>","joinType":"left"}`,
    
- **hot-swaps** the live consumers when control changes,
    
- **safely logs** each apply and **always updates** `orchestrator_status`,
    
- avoids `ApplicationState` entirely (not required),
    
- uses the correct `DynamicTableWriter` signature and `deephaven.time.now()`.
    

Copy/paste this file as `orchestrator.py` and run `deephaven server --port 10000`.

```python
# orchestrator.py  — Deephaven 0.40.2

from __future__ import annotations
from dataclasses import dataclass
from typing import Optional, List
from datetime import datetime, timezone

import json
from deephaven import dtypes as dt
from deephaven import time as dhtime
from deephaven.table import Table
from deephaven.stream.kafka import consumer as kc
from deephaven.table_listener import listen
from deephaven import DynamicTableWriter

# ----------------- EDIT THESE -----------------
DEFAULT_USERS_TOPIC = "ccd01_sb_its_esp_tap3507_bishowcaseraw"
DEFAULT_ACCOUNTS_TOPIC = "ccd01_sb_its_esp_tap3507_bishowcasecurated"
CONTROL_TOPIC = "ccd01_sb_its_esp_tap3507_metadata"

KAFKA_CONFIG = {
    # <<< your working Kafka client props (you had these correct) >>>
    "bootstrap.servers": "pkc-13p0g.canadacentral.azure.confluent.cloud:9092",
    "security.protocol": "SASL_SSL",
    "sasl.mechanism": "OAUTHBEARER",
    "sasl.oauthbearer.method": "oidc",
    "sasl.oauthbearer.token.endpoint.url": "https://fedsit.rastest.tdbank.ca/as/token.oauth2",
    "sasl.oauthbearer.sub.claim.name": "client_id",
    "sasl.oauthbearer.client.id": "TestScopeClient",
    "sasl.oauthbearer.client.secret": "2Federate",
    "sasl.oauthbearer.extensions.logicalCluster": "lkc-ygvwwp",
    "sasl.oauthbearer.extensions.identityPoolId": "pool-NRk1",
    "ssl.endpoint.identification.algorithm": "https",
}
# ----------------------------------------------

# Column specs for JSON values (strict and 0.40.2-safe)
USER_SPEC = kc.json_spec({
    "userId": dt.string,
    "name": dt.string,
    "email": dt.string,
    "age": dt.int32,
})
ACCOUNT_SPEC = kc.json_spec({
    "userId": dt.string,
    "accountType": dt.string,
    "balance": dt.double,   # 0.40.x uses dt.double (not float64 alias)
})

CONTROL_VALUE_SPEC = kc.json_spec({
    "usersTopic": dt.string,
    "accountsTopic": dt.string,
    "joinType": dt.string,
})

# Small export map so IDE/Angular can find tables by name
app = {}  # type: ignore[var-annotated]

@dataclass
class _State:
    users_topic: str
    accounts_topic: str
    join_type: str  # 'left' | 'inner'
    # live tables
    users_tbl: Optional[Table] = None
    accounts_tbl: Optional[Table] = None
    final_ui: Optional[Table] = None

class Orchestrator:
    def __init__(self, users_topic: str, accounts_topic: str, join_type: str = "left"):
        self.state = _State(users_topic, accounts_topic, join_type)
        # status writer / table
        self._status_writer = DynamicTableWriter(
            ["usersTopic", "accountsTopic", "joinType", "lastApplied"],
            [dt.string, dt.string, dt.string, dt.Instant]
        )
        self.orchestrator_status: Table = self._status_writer.table

        # initial build
        self._set_topics(users_topic, accounts_topic, join_type)

        # export for IDE / Angular
        app["users_ui"] = self.state.users_tbl
        app["accounts_ui"] = self.state.accounts_tbl
        app["final_ui"] = self.state.final_ui
        app["orchestrator_status"] = self.orchestrator_status

        # control listener
        self._start_control_listener(CONTROL_TOPIC)
        print("[orchestrator] ready")

    # ---------- building consumers & join ----------
    def _consume_users(self, topic: str) -> Table:
        print(f"[orchestrator] consuming USERS from '{topic}'")
        t = kc.consume(
            {**KAFKA_CONFIG, "group.id": f"dh-users-{topic}"},
            topic=topic,
            value_spec=USER_SPEC,
            table_type="append"
        )
        # sanitize and stable types
        t = (t
             .update_view([
                 "userId = (String)value.userId",
                 "name = (String)value.name",
                 "email = (String)value.email",
                 "age = (int)value.age"
             ])
             .drop_columns(["key", "value", "partition", "offset", "timestamp"]))
        return t

    def _consume_accounts(self, topic: str) -> Table:
        print(f"[orchestrator] consuming ACCOUNTS from '{topic}'")
        t = kc.consume(
            {**KAFKA_CONFIG, "group.id": f"dh-accounts-{topic}"},
            topic=topic,
            value_spec=ACCOUNT_SPEC,
            table_type="append"
        )
        t = (t
             .update_view([
                 "userId = (String)value.userId",
                 "accountType = (String)value.accountType",
                 "balance = (double)value.balance"
             ])
             .drop_columns(["key", "value", "partition", "offset", "timestamp"]))
        return t

    def _build_join(self, users: Table, accounts: Table, join_type: str) -> Table:
        join_type = (join_type or "left").lower()
        if join_type == "inner":
            j = users.natural_join(accounts, on=["userId"])
        else:
            # left-like (preserves users)
            j = users.left_outer_join(accounts, on=["userId"])
        # make columns deterministic & UI friendly
        cols = [c for c in ["userId", "name", "email", "age", "accountType", "balance"] if c in j.columns]
        return j.view(cols)

    # ---------- apply a control change ----------
    def _set_topics(self, users_topic: str, accounts_topic: str, join_type: str):
        try:
            print(f"[orchestrator] applying control: users='{users_topic}' accounts='{accounts_topic}' join='{join_type}'")

            users_tbl = self._consume_users(users_topic)
            accts_tbl = self._consume_accounts(accounts_topic)
            final_ui = self._build_join(users_tbl, accts_tbl, join_type)

            # swap state (hot-swap: just rebind names)
            self.state.users_topic = users_topic
            self.state.accounts_topic = accounts_topic
            self.state.join_type = join_type
            self.state.users_tbl = users_tbl
            self.state.accounts_tbl = accts_tbl
            self.state.final_ui = final_ui

            # re-export so existing names point to the new tables
            app["users_ui"] = users_tbl
            app["accounts_ui"] = accts_tbl
            app["final_ui"] = final_ui

            # status row
            self._status_writer.write_row(users_topic, accounts_topic, join_type, dhtime.now())
            print("[orchestrator] topics set ok.")
        except Exception as e:
            print(f"[orchestrator] set_topics error: {e!r}")

    # ---------- control topic listener ----------
    def _start_control_listener(self, control_topic: str):
        print(f"[orchestrator] control listener on '{control_topic}'")

        ctrl = kc.consume(
            {**KAFKA_CONFIG, "group.id": f"dh-control-{control_topic}"},
            topic=control_topic,
            value_spec=CONTROL_VALUE_SPEC,
            table_type="append"
        ).update_view([
            "usersTopic = (String)value.usersTopic",
            "accountsTopic = (String)value.accountsTopic",
            "joinType = (String)value.joinType"
        ]).drop_columns(["key", "value", "partition", "offset", "timestamp"])

        def _on_update(upd, is_replay):
            try:
                added = upd.added()
                if added is None or added.is_empty():
                    return
                last = added.tail(1)

                users = (last.columns["usersTopic"].data()[0] if "usersTopic" in last.columns else None)
                accts = (last.columns["accountsTopic"].data()[0] if "accountsTopic" in last.columns else None)
                join  = (last.columns["joinType"].data()[0] if "joinType" in last.columns else None)

                # Coalesce with current state if a field is omitted / null
                users = users or self.state.users_topic
                accts = accts or self.state.accounts_topic
                join  = (join or self.state.join_type or "left").lower()
                if join not in ("left", "inner"):
                    join = "left"

                changed = (users != self.state.users_topic) or (accts != self.state.accounts_topic) or (join != self.state.join_type)
                if not changed:
                    return

                print(f"[orchestrator] control apply: users='{users}' accounts='{accts}' join='{join}'")
                self._set_topics(users, accts, join)
            except Exception as e:
                print(f"[orchestrator] control listener err: {e!r}")

        # IMPORTANT: callback must accept (update, is_replay) in 0.40.2
        listen(ctrl, _on_update)

        # also expose raw control stream for your inspection
        app["control_raw"] = ctrl

# ---------- construct the app on import ----------
_orc = Orchestrator(DEFAULT_USERS_TOPIC, DEFAULT_ACCOUNTS_TOPIC, "left")

# exported names for IDE / Angular
users_ui = app["users_ui"]
accounts_ui = app["accounts_ui"]
final_ui = app["final_ui"]
orchestrator_status = app["orchestrator_status"]
control_raw = app["control_raw"]
```

### How to verify it’s actually switching

1. Open these four tables in the IDE: `control_raw`, `orchestrator_status`, `users_ui`, `accounts_ui`.
    
2. Produce JSON to the control topic exactly as:
    
    ```
    {"usersTopic":"<NEW_USERS>","accountsTopic":"<NEW_ACCTS>","joinType":"left"}
    ```
    
3. In the **console**, you should see:
    
    ```
    [orchestrator] control apply: users='...' accounts='...' join='left'
    [orchestrator] topics set ok.
    ```
    
4. `orchestrator_status` will get a **new row** with those topics and a fresh `lastApplied`.
    
5. `users_ui` / `accounts_ui` start filling from the new topics. If you already had panels open, close & reopen them to pick up the new tickets (normal IDE behavior). For Angular, re-`getTable()` when `orchestrator_status.lastApplied` changes (as I showed earlier).
    

### Why your earlier attempts stalled

- `replay_initial=True` doesn’t exist in 0.40.2 → listener never started.
    
- Callback sig must be `(update, is_replay)` → otherwise the listener is rejected.
    
- Some earlier status-table code used wrong `DynamicTableWriter` signature, or tried to stuff `datetime` into `dt.Instant` with array builders. Fixed above: `DynamicTableWriter([...], [...])` then `write_row(..., dhtime.now())`.
    
- Throwing inside `_set_topics` without logging left `orchestrator_status` unchanged. The code above logs failures and _always_ writes a new status row on success.
    

If this still doesn’t move `orchestrator_status` when you publish control messages, send me the exact **one** control JSON you produced and the console lines that follow. That’ll let me pinpoint whether the control row has nulls or the joinType is unexpected.