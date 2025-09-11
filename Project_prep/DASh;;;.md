


---

# 1) Deephaven ↔ Kafka connectivity (OAUTHBEARER)

> Put your actual values in ALL_CAPS placeholders. Keep secrets in env vars if possible.

```python
# In Deephaven's Python console (or a startup script)
from deephaven import kafka, dtypes as dht

# --- If you store secrets in env vars (recommended) ---
import os
KAFKA_BOOTSTRAP = os.getenv("KAFKA_BOOTSTRAP", "pkc-...:9092")
KAFKA_TOPIC     = os.getenv("KAFKA_TOPIC", "gold_positions_changes")

kprops = {
    # Core
    "bootstrap.servers": KAFKA_BOOTSTRAP,
    "group.id": "dh-gold-positions-consumer",
    "auto.offset.reset": "latest",
    "enable.auto.commit": "true",

    # Security: SASL over TLS
    "security.protocol": "SASL_SSL",
    "sasl.mechanism": "OAUTHBEARER",

    # ----- OAUTH settings -----
    # NOTE: One of these class names will work depending on how Kafka is packaged in your Deephaven build.
    # Try the non-shaded name first; if you see ClassNotFound, switch to the 'kafkashaded...' version.
    "sasl.login.callback.handler.class":
        "org.apache.kafka.common.security.oauthbearer.secured.OAuthBearerLoginCallbackHandler",
        # "kafkashaded.org.apache.kafka.common.security.oauthbearer.secured.OAuthBearerLoginCallbackHandler",

    "sasl.oauthbearer.token.endpoint.url": "https://FEDSIT_OR_YOUR_IDP/oauth2/token",

    # Client creds (use env vars/secrets in practice)
    "sasl.oauthbearer.client.id":     os.getenv("KAFKA_CLIENT_ID",     "YOUR_CLIENT_ID"),
    "sasl.oauthbearer.client.secret": os.getenv("KAFKA_CLIENT_SECRET", "YOUR_CLIENT_SECRET"),

    # Optional but often required fields; mirror what you set in Databricks:
    # Claim to map as the principal (matches your Databricks "sasl.oauthbearer.sub.claim.name")
    "sasl.oauthbearer.sub.claim.name": "client_id",

    # If your broker expects custom OAuth "extensions" (you had these in your notebook):
    "sasl.oauthbearer.extensions.logicalCluster":  os.getenv("LOGICAL_CLUSTER",  "lkc-xxxxx"),
    "sasl.oauthbearer.extensions.identityPoolId":  os.getenv("IDENTITY_POOL_ID", "pool-xxxxx"),
    "sasl.oauthbearer.extensions.identityPool":    os.getenv("IDENTITY_POOL",    "poolName"),

    # TLS hostname verification (keep https unless your infra says otherwise)
    "ssl.endpoint.identification.algorithm": "https",

    # Deserializers – if your producer sends JSON text, StringDeserializer is best:
    "key.deserializer":   "org.apache.kafka.common.serialization.StringDeserializer",
    "value.deserializer": "org.apache.kafka.common.serialization.StringDeserializer",
}

# Consume JSON directly into a refreshing table
t = kafka.consume(
    properties=kprops,
    topic=KAFKA_TOPIC,
    key_format="string",   # adjust if you're not sending a key
    value_format="json",   # expects each Kafka message value to be a JSON object
    table_type="append",   # "append" (keep all rows) or "stream" (blink table)
    # Optional explicit schema (faster & safer than schema inference):
    value_fields=[
        ("event_ts", dht.Instant),
        ("symbol",   dht.string),
        ("price",    dht.double),
        ("qty",      dht.int_),
        ("op",       dht.string),    # 'I','U','D' if you’re sending CDC-style events
        # ...add the other columns you emit from Databricks...
    ],
)

# If you have a structured key (e.g., symbol+date) and you're sending it as JSON,
# set key_format="json" and specify key_fields=[(...)] above.
```

### Notes / common gotchas

- **Class name** for `sasl.login.callback.handler.class`:
    
    - Try `org.apache.kafka.common.security.oauthbearer.secured.OAuthBearerLoginCallbackHandler` first.
        
    - If you get `ClassNotFound`, switch to the **shaded** name:  
        `kafkashaded.org.apache.kafka.common.security.oauthbearer.secured.OAuthBearerLoginCallbackHandler`.
        
- **Value format**: If Databricks writes JSON **strings**, the `StringDeserializer` + `value_format="json"` is correct.  
    If you somehow write **base64-encoded** bytes, switch to `ByteArrayDeserializer` and decode in Deephaven before parsing.
    
- **Event Hubs** users typically use SASL/PLAIN—not OAUTHBEARER. Your screenshot looks like a proper OAuth/OIDC broker, so keep OAUTHBEARER.
    

---

# 2) Turn the stream into a “materialized” live view (upserts, windows, KPIs)

Assuming your JSON includes CDC fields (`op`, keys, timestamps):

```python
from deephaven import time_table, agg as aggby

# 2a) Materialize latest row per business key (e.g., symbol)
#     Replace ["symbol"] with your true key(s): ["cusip","tradingDate"], etc.
latest = t.sort_descending("event_ts").last_by("symbol")

# 2b) Rolling window metrics (e.g., last 5 minutes, per symbol)
#     Convert event_ts to Deephaven Instant if it arrives as string:
live = t.update_view([
    "ts = (Instant)event_ts",      # drop if event_ts already Instant
    "px = (double)price",
    "qty_i = (int)qty"
])

# 5-minute tumbling count/avg per symbol (updates continuously)
kpis_5m = live.where("ts >= now() - MINUTE * 5") \
              .group_by("symbol") \
              .update_view([
                  "trades = size()", 
                  "avg_px = avg(px)", 
                  "qty_sum = sum(qty_i)"
              ])

# 2c) Top movers snapshot
top_by_px = latest.sort_descending("price").head(20)
```

> Any table above (`t`, `latest`, `kpis_5m`, `top_by_px`) is **refreshing**. Point charts/grids at them and they’ll update in place.

---

# 3) (Optional) Join “hot” stream with “cold” history

If you have historical Parquet/Delta extracts locally or on a mounted path, read once and join:

```python
from deephaven import parquet as dhpq

hist = dhpq.read("/mnt/data/positions_history.parquet")  # non-refreshing
enriched = latest.natural_join(hist, on=["symbol"], joins=["sector","exchange"])
```

---

# 4) Quick sanity checklist

- **Can Deephaven reach the brokers?** From your laptop, test TCP 9092/9093 or the load balancer DNS; if it’s private, use VPN/ZTNA.
    
- **Auth errors** → check client_id/secret, token endpoint URL, and the callback handler class.
    
- **No rows** → confirm the **topic name**, and that your producer is writing **JSON text** (or adjust deserializers).
    
- **Schema drift** → list only stable `value_fields` and add new ones later.
    

---

If you share:

- the **topic name**,
    
- a sample **JSON message**, and
    
- your **business keys** (e.g., `["cusip","tradingDate"]`),
    

I’ll tailor the `consume(...)` call (schema + key handling) and drop in a ready-made **dashboard table bundle** (latest snapshot, rolling 5/15-minute KPIs, and top-N) that you can use immediately.


-----------------------------------------


Great—use Deephaven’s **app.d** autoload folder. Anything you drop there (Python files) is executed when the server starts, so your Kafka reader + live tables are ready the moment you open the IDE.

Here’s a clean, copy-paste setup for both Windows (PowerShell) and Linux/WSL.

---

# 1) Pick folders & point Deephaven to them

## Windows (PowerShell)

```powershell
# choose a home
$DH_HOME="C:\deephaven"

# create folders
New-Item -Force -ItemType Directory "$DH_HOME\data"        | Out-Null
New-Item -Force -ItemType Directory "$DH_HOME\config\app.d" | Out-Null

# make these available for the current shell/session (persist them in System Env Var if you like)
$env:DEEPHAVEN_DATA_DIR   = "$DH_HOME\data"
$env:DEEPHAVEN_CONFIG_DIR = "$DH_HOME\config"
```

## Linux / WSL (bash)

```bash
export DH_HOME="$HOME/deephaven"
mkdir -p "$DH_HOME/data" "$DH_HOME/config/app.d"
export DEEPHAVEN_DATA_DIR="$DH_HOME/data"
export DEEPHAVEN_CONFIG_DIR="$DH_HOME/config"
```

> Deephaven will auto-scan **`$DEEPHAVEN_CONFIG_DIR/app.d`** (and also `$DEEPHAVEN_DATA_DIR/app.d` if you prefer). Use either; I’ll use **config/app.d** below.

---

# 2) Put your Kafka consumer script in `app.d`

Create:  
**`$DEEPHAVEN_CONFIG_DIR/app.d/consume_gold_kafka.py`**

```python
# consume_gold_kafka.py
from deephaven import kafka, dtypes as dht

# --- pull secrets from env when possible ---
import os
BOOTSTRAP = os.getenv("KAFKA_BOOTSTRAP", "pkc-...:9092")
TOPIC     = os.getenv("KAFKA_TOPIC", "gold_positions_changes")

kprops = {
    "bootstrap.servers": BOOTSTRAP,
    "group.id": "dh-gold-positions-consumer",
    "auto.offset.reset": "latest",

    # Security: SASL over TLS with OAuthBearer (matches your Databricks config)
    "security.protocol": "SASL_SSL",
    "sasl.mechanism": "OAUTHBEARER",
    # If ClassNotFound occurs, switch to the shaded name (uncomment next line, comment the current one)
    "sasl.login.callback.handler.class":
        "org.apache.kafka.common.security.oauthbearer.secured.OAuthBearerLoginCallbackHandler",
        # "kafkashaded.org.apache.kafka.common.security.oauthbearer.secured.OAuthBearerLoginCallbackHandler",

    "sasl.oauthbearer.token.endpoint.url": os.getenv("OAUTH_TOKEN_URL", "https://<your-idp>/oauth2/token"),
    "sasl.oauthbearer.client.id":     os.getenv("KAFKA_CLIENT_ID",     "<client_id>"),
    "sasl.oauthbearer.client.secret": os.getenv("KAFKA_CLIENT_SECRET", "<client_secret>"),
    "sasl.oauthbearer.sub.claim.name": os.getenv("OAUTH_SUB_CLAIM", "client_id"),

    # If your broker expects the extra OAuth extensions you showed
    "sasl.oauthbearer.extensions.logicalCluster": os.getenv("LOGICAL_CLUSTER",  "lkc-xxxxx"),
    "sasl.oauthbearer.extensions.identityPoolId": os.getenv("IDENTITY_POOL_ID", "pool-xxxxx"),
    "sasl.oauthbearer.extensions.identityPool":   os.getenv("IDENTITY_POOL",    "poolName"),

    "ssl.endpoint.identification.algorithm": "https",

    # Deserialize JSON text
    "key.deserializer":   "org.apache.kafka.common.serialization.StringDeserializer",
    "value.deserializer": "org.apache.kafka.common.serialization.StringDeserializer",
}

# Build a refreshing table straight from Kafka (declare the schema you emit)
t = kafka.consume(
    properties=kprops,
    topic=TOPIC,
    key_format="string",
    value_format="json",
    table_type="append",   # or "stream" for blink
    value_fields=[
        ("event_ts", dht.Instant),  # ensure your JSON value has fields with these names/types
        ("symbol",   dht.string),
        ("price",    dht.double),
        ("qty",      dht.int_),
        ("op",       dht.string),
        # ... add any other columns from your Databricks JSON
    ],
)

# Materialized latest row per symbol for dashboards
latest = t.sort_descending("event_ts").last_by("symbol")

# A small KPI table (rolling 5 min)
live = t.update_view([
    "ts = (Instant)event_ts",
    "px = (double)price",
    "qty_i = (int)qty",
])
kpi_5m = live.where("ts >= now() - MINUTE * 5") \
             .group_by("symbol") \
             .update_view([
                 "trades = size()", 
                 "avg_px = avg(px)", 
                 "qty_sum = sum(qty_i)"
             ])

# Optionally “publish” friendly names to find easily in the IDE
from deephaven.server import ServerContext
ServerContext.publish_table("gold_stream_raw", t)
ServerContext.publish_table("gold_latest", latest)
ServerContext.publish_table("gold_kpi_5m", kpi_5m)
```

> If your broker uses a **shaded** Kafka in Deephaven and you get a `ClassNotFound` on the callback handler, flip the class name to the `kafkashaded...` one in the comment.

---

# 3) (Optional) Set env vars for secrets before launch

## Windows (PowerShell)

```powershell
$env:KAFKA_BOOTSTRAP="pkc-...:9092"
$env:KAFKA_TOPIC="gold_positions_changes"
$env:KAFKA_CLIENT_ID="..."
$env:KAFKA_CLIENT_SECRET="..."
$env:OAUTH_TOKEN_URL="https://fedsit.ras.../oauth2/token"
$env:LOGICAL_CLUSTER="lkc-xxxxx"
$env:IDENTITY_POOL_ID="pool-xxxxx"
$env:IDENTITY_POOL="poolName"
```

## Linux / WSL (bash)

```bash
export KAFKA_BOOTSTRAP="pkc-...:9092"
export KAFKA_TOPIC="gold_positions_changes"
export KAFKA_CLIENT_ID="..."
export KAFKA_CLIENT_SECRET="..."
export OAUTH_TOKEN_URL="https://fedsit.ras.../oauth2/token"
export LOGICAL_CLUSTER="lkc-xxxxx"
export IDENTITY_POOL_ID="pool-xxxxx"
export IDENTITY_POOL="poolName"
```

---

# 4) Start the server and open the IDE

```bash
deephaven-server --port 10000
```

Open **[http://localhost:10000/ide](http://localhost:10000/ide)** → in the **Variables** pane you should see:

- `gold_stream_raw`
    
- `gold_latest`
    
- `gold_kpi_5m`
    

Click any to view; build charts from them (they’re **refreshing**).

---

## Quick dashboard example (drop this into a new IDE Python cell)

```python
from deephaven.plot import Figure

# Price by symbol — live line chart
fig_price = (Figure()
  .plot_xy(series_name="price", t=gold_stream_raw.update_view(["ts=(Instant)event_ts","px=(double)price"]), x="ts", y="px", by=["symbol"])
  .show())

# Top 20 by latest price — live table
top20 = gold_latest.sort_descending("price").head(20)
```

---

## Troubleshooting tips

- **Auth error** → check `client_id/secret`, token URL, and try the shaded vs non-shaded callback handler class.
    
- **No data** → confirm the **topic name** and that your producer writes JSON text. If you produce Base64 bytes, swap to `ByteArrayDeserializer` and parse in Python.
    
- **Connectivity** → if brokers are in a private VNet, connect your laptop via VPN/ZTNA first.
    

If you paste one **sample JSON message** from your topic and your **true key columns**, I can align the `value_fields` and the “latest by” logic precisely for your schema.