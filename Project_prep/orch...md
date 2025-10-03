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