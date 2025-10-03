Perfect, thanks for sharing that error screenshot. The key line is:

```
AttributeError: module 'jpy' has no attribute 'JProxy'
```

👉 That means your Deephaven build’s `jpy` binding doesn’t support `JProxy`. Some builds of Deephaven ship `jpy` without the proxy helpers, so you can’t create a Java interface implementation from Python that way.

---

### ✅ How to fix

We’ll avoid `JProxy` entirely. Deephaven actually provides a **built-in mechanism** for “listen to a table and react” using Python: **`update_graph` hooks**.

Here’s how you can rewrite your listener logic without `JProxy`:

```python
from deephaven.stream.kafka import consumer as kc
from deephaven import dtypes as dt, empty_table
from deephaven.experimental.outer_joins import left_outer_join

# (keep your KAFKA_CONFIG, USER_VALUE_SPEC, ACCOUNT_VALUE_SPEC, CONTROL_SPEC as before)

# --- Helpers ---
def _err_table(msg: str):
    return empty_table(1).update_view([f"msg=`{msg}`"])

def _count_table(t):
    return t.update_view(["one=1"]).cum_sum(cols=["one"]).view(["rows=one"]).tail(1)

# --- UI placeholders ---
first_scope_ui  = empty_table(0)
second_scope_ui = empty_table(0)
final_ui        = empty_table(0)
apply_status    = _err_table("idle")
apply_err       = _err_table("")

# --- join selector ---
def _join_by_type(jtype: str, lhs, rhs, on_cols, join_cols):
    jt = (jtype or "LEFT_OUTER").upper()
    if jt == "LEFT_OUTER":
        return left_outer_join(lhs, rhs, on=on_cols, joins=join_cols)
    if jt == "NATURAL":
        return lhs.natural_join(rhs, on=on_cols, joins=join_cols)
    if jt == "EXACT":
        return lhs.exact_join(rhs, on=on_cols, joins=join_cols)
    raise ValueError(f"Unknown joinType: {jtype}")

# --- apply logic ---
def _apply(topicA: str, topicB: str, joinType: str):
    global first_scope_ui, second_scope_ui, final_ui, apply_status, apply_err

    try:
        apply_status = _err_table(f"building {topicA}, {topicB}, {joinType}")
        DATA_CFG = dict(KAFKA_CONFIG)
        DATA_CFG["auto.offset.reset"] = "earliest"

        left_raw = kc.consume(DATA_CFG, topicA,
            key_spec=kc.KeyValueSpec.IGNORE,
            value_spec=USER_VALUE_SPEC,
            table_type=kc.TableType.append(),
        )
        right_raw = kc.consume(DATA_CFG, topicB,
            key_spec=kc.KeyValueSpec.IGNORE,
            value_spec=ACCOUNT_VALUE_SPEC,
            table_type=kc.TableType.append(),
        )

        A = left_raw.view(["userId", "name", "email", "age"])
        B = right_raw.view(["userId", "accountType", "balance"])
        J = _join_by_type(joinType, A, B, ["userId"], ["accountType","balance"]) \
                .view(["userId","accountType","balance","name","email","age"])

        first_scope_ui, second_scope_ui, final_ui = A, B, J
        apply_err    = _err_table("")
        apply_status = _err_table("OK")
    except Exception as e:
        apply_err    = _err_table(str(e))
        apply_status = _err_table("ERR")

# --- control consumer ---
CTRL_CFG = dict(KAFKA_CONFIG)
CTRL_CFG["auto.offset.reset"] = "earliest"

control_raw = kc.consume(CTRL_CFG, CONTROL_TOPIC,
    key_spec=kc.KeyValueSpec.IGNORE,
    value_spec=CONTROL_SPEC,
    table_type=kc.TableType.append(),
)

control_latest = control_raw.last_by(["topicA","topicB","joinType"]).view(["topicA","topicB","joinType"])

# --- auto-apply on control updates using update_graph ---
def _on_control_update(t, is_replay):
    try:
        row = t.last_row
        if row is None: 
            return
        topicA, topicB, joinType = row["topicA"], row["topicB"], row["joinType"]
        if topicA and topicB:
            _apply(topicA, topicB, joinType)
    except Exception as e:
        apply_err = _err_table(str(e))

control_latest.update_graph(_on_control_update)
```

---

### 🔑 Key differences

- Instead of `jpy.JProxy` and `addUpdateListener`, we just use:
    
    ```python
    control_latest.update_graph(_on_control_update)
    ```
    
    That attaches a Python callback that fires every time `control_latest` ticks.
    
- Inside `_on_control_update`, we read the last row and call `_apply`.
    
- No need for `QueryScope`, `query_library`, or `JProxy`.
    

---

### Next steps for you

1. Replace your listener section with the `update_graph` approach above.
    
2. Produce a control message into your `CONTROL_TOPIC` (with `{topicA, topicB, joinType}`).
    
3. Check:
    
    - `control_latest` → updated values
        
    - `apply_status` → “OK”
        
    - `first_scope_ui`, `second_scope_ui`, `final_ui` → streaming tables ticking from new topics
        

---

Would you like me to **re-assemble your full script** (with KAFKA_CONFIG + specs + updated listener) so you have a clean one-piece code drop-in?

------------

You’re seeing:

```
TypeError: 'updategraph' object is not callable
```

That’s because in your Deephaven build `update_graph` is **not** a callable function on tables. Instead, use the **table listener helper** to react to updates. Below is a full, drop-in script that:

- Consumes a **control topic** (`topicA`, `topicB`, `joinType`)
    
- Rebuilds the two Kafka consumers + the join whenever a new control row arrives
    
- Keeps three **streaming** output tables always available: `first_scope_ui`, `second_scope_ui`, `final_ui`
    
- Surfaces status/errors in `apply_status` and `apply_err`
    
- Includes a `control_debug` tail for quick inspection
    

It avoids `QueryScope`, `query_library`, and `jpy.JProxy` entirely.

---

### ✅ Full updated script (ready to run)

```python
# --- Imports ---------------------------------------------------------------
from deephaven.stream.kafka import consumer as kc
from deephaven import dtypes as dt, empty_table
from deephaven.experimental.outer_joins import left_outer_join
from deephaven.table_listener import listen  # <-- use listener helper

# --- Topics ----------------------------------------------------------------
# control topic is produced by your Spring Boot app
CONTROL_TOPIC = "ccd01_sb_its_esp_tap3507_metadata"

# --- Kafka config ----------------------------------------------------------
KAFKA_CONFIG = {
    "bootstrap.servers": "pkc-lk130p.canadacentral.azure.confluent.cloud:9092",
    "auto.offset.reset": "latest",
    "security.protocol": "SASL_SSL",
    "sasl.mechanism": "OAUTHBEARER",
    "sasl.login.callback.handler.class":
        "org.apache.kafka.common.security.oauthbearer.secured.OAuthBearerLoginCallbackHandler",
    "sasl.oauthbearer.extensions.scope.claim.name": "client_id",
    "sasl.oauthbearer.client.id": "TestScopeClient",
    "sasl.oauthbearer.client.secret": "2Federate",
    "scope": "",
    "sasL.oauthbearer.extensions.logicalCluster": "lkc-ygywwp",
    "sasl.oauthbearer.extensions.identityPoolId": "pool-NRk1",
    "sasl.oauthbearer.token.endpoint.url": "https://fedsit.rastest.tdbank.ca/as/token.oauth2",
    "ssl.endpoint.identification.algorithm": "https",
}

# --- Value specs (schemas) -------------------------------------------------
USER_VALUE_SPEC = kc.json_spec({
    "userId": dt.string,
    "name":   dt.string,
    "email":  dt.string,
    "age":    dt.int64
})

ACCOUNT_VALUE_SPEC = kc.json_spec({
    "userId":      dt.string,
    "accountType": dt.string,
    "balance":     dt.double
})

CONTROL_SPEC = kc.json_spec({
    "topicA":   dt.string,
    "topicB":   dt.string,
    "joinType": dt.string,   # "LEFT_OUTER" | "NATURAL" | "EXACT"
})

# --- Small helpers ---------------------------------------------------------
def _err_table(msg: str):
    return empty_table(1).update_view([f"msg=`{msg}`"])

# Join selector: use experimental left_outer_join; NATURAL/EXACT via table methods
def _join_by_type(jtype: str, lhs, rhs, on_cols, join_cols):
    jt = (jtype or "LEFT_OUTER").upper()
    if jt == "LEFT_OUTER":
        return left_outer_join(lhs, rhs, on=on_cols, joins=join_cols)
    if jt == "NATURAL":
        return lhs.natural_join(rhs, on=on_cols, joins=join_cols)
    if jt == "EXACT":
        return lhs.exact_join(rhs, on=on_cols, joins=join_cols)
    raise ValueError(f"Unknown joinType: {jtype}")

# --- UI placeholders (always exist so Angular can bind immediately) --------
first_scope_ui  = empty_table(0)
second_scope_ui = empty_table(0)
final_ui        = empty_table(0)

apply_status = _err_table("idle")
apply_err    = _err_table("")

# --- Rebuild data consumers + join ----------------------------------------
def _apply(topicA: str, topicB: str, joinType: str):
    """
    Recreate A/B Kafka append tables and rebuild the selected join.
    Rebinds first_scope_ui / second_scope_ui / final_ui globals.
    """
    global first_scope_ui, second_scope_ui, final_ui, apply_status, apply_err

    try:
        apply_status = _err_table(f"building: {topicA} , {topicB} , {joinType}")

        # Read earliest so we have rows as soon as we switch topics
        DATA_CFG = dict(KAFKA_CONFIG)
        DATA_CFG["auto.offset.reset"] = "earliest"

        left_raw = kc.consume(
            DATA_CFG, topicA,
            key_spec=kc.KeyValueSpec.IGNORE,
            value_spec=USER_VALUE_SPEC,
            table_type=kc.TableType.append(),
        )

        right_raw = kc.consume(
            DATA_CFG, topicB,
            key_spec=kc.KeyValueSpec.IGNORE,
            value_spec=ACCOUNT_VALUE_SPEC,
            table_type=kc.TableType.append(),
        )

        A = left_raw.view(["userId", "name", "email", "age"])
        B = right_raw.view(["userId", "accountType", "balance"])

        J = _join_by_type(joinType, A, B, ["userId"], ["accountType", "balance"]) \
                .view(["userId", "accountType", "balance", "name", "email", "age"])

        first_scope_ui, second_scope_ui, final_ui = A, B, J
        apply_err    = _err_table("")
        apply_status = _err_table("OK")
    except Exception as e:
        apply_err    = _err_table(str(e))
        apply_status = _err_table("ERR")

# --- Control stream (what Spring Boot produces) ----------------------------
CTRL_CFG = dict(KAFKA_CONFIG)
CTRL_CFG["auto.offset.reset"] = "earliest"

control_raw = kc.consume(
    CTRL_CFG, CONTROL_TOPIC,
    key_spec=kc.KeyValueSpec.IGNORE,
    value_spec=CONTROL_SPEC,
    table_type=kc.TableType.append(),
)

# Useful inspection tables in Panels
control_debug  = control_raw.tail(10)
control_latest = control_raw.last_by(["topicA", "topicB", "joinType"]).view(["topicA", "topicB", "joinType"])

# --- Listener: react to control updates (no QueryScope, no jpy) ------------
def _on_control_update(update, is_replay):
    """
    Called by Deephaven whenever control_latest updates.
    """
    try:
        # Get the newest row values safely via a tiny pandas hop
        pdf = control_latest.to_pandas()
        if not pdf.empty:
            topicA, topicB, joinType = pdf.iloc[-1].tolist()
            if isinstance(topicA, str) and isinstance(topicB, str):
                _apply(topicA, topicB, joinType or "LEFT_OUTER")
    except Exception as e:
        # surface listener errors without crashing the graph
        global apply_err, apply_status
        apply_err    = _err_table(f"listener: {e}")
        apply_status = _err_table("ERR")

# Attach the listener (this replaces any JProxy / QueryScope approach)
ctrl_listener = listen(
    table=control_latest,
    on_update=_on_control_update,
    on_failure=lambda ex: None,
)

# Optional: if a row already exists when script starts, apply once at startup
try:
    pdf0 = control_latest.to_pandas()
    if not pdf0.empty:
        a0, b0, j0 = pdf0.iloc[-1].tolist()
        if isinstance(a0, str) and isinstance(b0, str):
            _apply(a0, b0, j0 or "LEFT_OUTER")
except Exception as e:
    apply_err    = _err_table(f"initial: {e}")
    apply_status = _err_table("ERR")
```

---

### How to debug quickly

- **control_debug** – shows the last 10 control messages (exact payload your Spring app sent).
    
- **control_latest** – one-row table with the _current_ `topicA`, `topicB`, `joinType`.
    
- **apply_status** – `"OK"` after a successful rebuild; `"ERR"` if something failed; `"idle"` before first apply.
    
- **apply_err** – the error message (if any) from the last build attempt.
    
- **first_scope_ui / second_scope_ui / final_ui** – your three **streaming** result tables.
    

If first/second/final are empty, check:

1. `control_latest` has strings for `topicA` and `topicB` (no `null`).
    
2. Those topics actually have data and your JSON schema matches the messages.
    
3. `apply_status` is `OK`. If `ERR`, open `apply_err` to see why.
    

---

If you want these tables to **rebuild automatically** every time the Spring Boot app writes a new `control` message, you don’t need to call anything from Angular—this listener already does it.