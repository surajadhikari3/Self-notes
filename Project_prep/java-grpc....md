Got it — I can see that your `create_live_table()` function successfully consumes your Kafka topic and creates a live Deephaven table, but you’re seeing **extra metadata columns** like:

```
KafkaPartition, KafkaOffset, KafkaTimestamp, KafkaKey
```

These columns are automatically added by `deephaven.stream.kafka.consumer.consume()` unless you **explicitly disable or filter** them.

---

### ✅ To drop or avoid Kafka metadata columns

You have two clean options:

---

### **Option 1: Disable metadata at the consumer level**

You can tell Deephaven to **not include** metadata columns in your config itself by adding these properties before calling `kc.consume()`:

```python
config.update({
    "deephaven.kafka.skip.offset.column": "true",
    "deephaven.kafka.skip.partition.column": "true",
    "deephaven.kafka.skip.timestamp.column": "true",
    "deephaven.kafka.skip.key.column": "true"
})
```

So your final config would look like this (just add those at the end):

```python
config = {
    "bootstrap.servers": "pkc-l3k9.canadacentral.azure.confluent.cloud:9092",
    "security.protocol": "SASL_SSL",
    "sasl.mechanism": "OAUTHBEARER",
    "sasl.login.callback.handler.class": "org.apache.kafka.common.security.oauthbearer.secured.OAuthBearerLoginCallbackHandler",
    "sasl.jaas.config": "org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required "
                        "clientId='TestScopeClient' "
                        "clientSecret='2Federate' "
                        "scope='lkc-ygywwp' "
                        "extension_logicalCluster='lkc-ygywwp' "
                        "extension_identityPoolId='pool-NRk1' "
                        "sasl.oauthbearer.token.endpoint.url='https://fedsit.rastest.tdbank.ca/as/token.oauth2';",
    "auto.offset.reset": "latest",
    "sasl.login.callback.handler.class": "org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginCallbackHandler",
    "sasl.oauthbearer.token.endpoint.url": "https://fedsit.rastest.tdbank.ca/as/token.oauth2",
    "sasl.oauthbearer.client.id": "TestScopeClient",
    "sasl.oauthbearer.client.secret": "2Federate",
    "sasl.oauthbearer.scope": "lkc-ygywwp",
    "sasl.oauthbearer.extensions.logicalCluster": "lkc-ygywwp",
    "sasl.oauthbearer.extensions.identityPoolId": "pool-NRk1",
    "sasl.endpoint.identification.algorithm": "https",

    # 👇 Add these
    "deephaven.kafka.skip.offset.column": "true",
    "deephaven.kafka.skip.partition.column": "true",
    "deephaven.kafka.skip.timestamp.column": "true",
    "deephaven.kafka.skip.key.column": "true"
}
```

---

### **Option 2: Drop columns after consuming**

If you prefer to still create the full table and then strip out Kafka metadata, just do:

```python
tbl = kc.consume(
    config=config,
    topic=topic,
    key_spec=k_spec,
    value_spec=v_spec,
    table_type=ttype,
)

# Drop Kafka metadata columns if they exist
drop_cols = [c for c in tbl.columns if c.startswith("Kafka")]
tbl = tbl.dropColumns(*drop_cols)
```

---

### ✅ Recommended approach

Use **Option 1** (consumer-level skip flags) because:

- It avoids unnecessary memory overhead.
    
- Keeps your tables clean right from ingestion.
    
- No need for a second transformation step.
    

---

Would you like me to update your full `create_live_table()` function with these flags added and formatted properly for copy-paste?

----------------


Awesome—here’s a drop-in, copy-paste update of your `create_live_table()` that **suppresses Kafka metadata at the source** (via Deephaven skip flags) and also does a defensive post-consume drop just in case your server ignores the flags.

```python
from deephaven import dtypes as dht
from deephaven.stream.kafka import consumer as kc
import __main__  # so we can publish to console scope

def create_live_table(
    topic: str,
    schema: dict[str, dht.Type],
    alias: str | None = None,
    config: dict | None = None,       # pass your OAuth/Kafka config here (see example below)
    table_type: str = "append",
    ignore_key: bool = True
) -> str:
    """
    Consumes a Kafka topic into a Deephaven live table, without Kafka metadata columns.
    - topic: Kafka topic name
    - schema: {column_name: dtypes.*} for the JSON payload (value)
    - alias: optional table name to publish into console/__main__
    - config: base Kafka config (bootstrap, security, OAuth, etc.)
    - table_type: "append" (default) or "blink"
    - ignore_key: if True, skip key column entirely
    """
    if not topic:
        raise ValueError("topic is required")

    name = alias or topic.replace("-", "_")
    if name in globals():
        print(f"[DH] Reusing table {name}")
        return name

    # Build config and add Deephaven flags to skip metadata columns at the source
    cfg = dict(config or {})
    cfg.setdefault("auto.offset.reset", "latest")
    cfg.setdefault("deephaven.kafka.skip.offset.column", "true")
    cfg.setdefault("deephaven.kafka.skip.partition.column", "true")
    cfg.setdefault("deephaven.kafka.skip.timestamp.column", "true")
    cfg.setdefault("deephaven.kafka.skip.key.column", "true")

    # Specs
    v_spec = kc.json_spec(schema)
    k_spec = kc.KeyValueSpec.IGNORE if ignore_key else None
    ttype = kc.TableType.Append if table_type.lower() == "append" else kc.TableType.Blink

    # Consume
    tbl = kc.consume(
        cfg,
        topic,
        key_spec=k_spec,
        value_spec=v_spec,
        table_type=ttype,
    )

    # Defensive cleanup in case server ignores skip flags (older builds/configs)
    for col in ("KafkaPartition", "KafkaOffset", "KafkaTimestamp", "KafkaKey"):
        if col in tbl.columns:
            tbl = tbl.dropColumns(col)

    # Publish for console/other scripts
    globals()[name] = tbl
    setattr(__main__, name, tbl)

    print(f"[DH] Created {name} from topic '{topic}'")
    return name
```

### Example `config` you pass in (trim to your environment)

```python
oauth_cfg = {
    "bootstrap.servers": "pkc-l3k9.canadacentral.azure.confluent.cloud:9092",
    "security.protocol": "SASL_SSL",
    "sasl.mechanism": "OAUTHBEARER",
    "sasl.login.callback.handler.class": "org.apache.kafka.common.security.oauthbearer.secured.OAuthBearerLoginCallbackHandler",
    "sasl.jaas.config":
        "org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required "
        "clientId='TestScopeClient' "
        "clientSecret='2Federate' "
        "scope='lkc-ygywwp' "
        "extension_logicalCluster='lkc-ygywwp' "
        "extension_identityPoolId='pool-NRk1' "
        "sasl.oauthbearer.token.endpoint.url='https://fedsit.rastest.tdbank.ca/as/token.oauth2';",
    "sasl.endpoint.identification.algorithm": "https",
    # The skip flags are also added in the function, but keeping them here is fine too:
    # "deephaven.kafka.skip.offset.column": "true",
    # "deephaven.kafka.skip.partition.column": "true",
    # "deephaven.kafka.skip.timestamp.column": "true",
    # "deephaven.kafka.skip.key.column": "true",
}
```

### Call

```python
from deephaven import dtypes as dht

employee_schema = {
    "employeeId": dht.string,
    "firstName": dht.string,
    "lastName": dht.string,
    "salary": dht.double,
    "eventTime": dht.Instant
}

create_live_table(
    topic="ccd01_sb_its_esp_tap3507_employee",
    schema=employee_schema,
    alias="employee",
    config=oauth_cfg,
    table_type="append",
    ignore_key=True
)
```

This will create `employee` **without** `KafkaPartition/KafkaOffset/KafkaTimestamp/KafkaKey`. If you want to keep _only_ timestamp for debugging, just remove the corresponding skip flag and the defensive drop for that single column.