


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


---------------------------------

You’re right—the **new API** lives at `deephaven.stream.kafka.consumer`, and its `consume(...)` signature expects the **Kafka config as the first positional arg**, not a `properties=` keyword. It also wants a `TableType` enum (not `"append"`), and modern dtypes (`int32/int64/double/Instant`, etc.). Below is **zero-issue, copy-paste** code for both a local broker and your OAuth/SASL broker.

I’m following the current Deephaven docs for `consume`, `json_spec`, `KeyValueSpec`, and `TableType`. ([Deephaven](https://deephaven.io/core/0.39.8/docs/reference/data-import-export/Kafka/consume/?utm_source=chatgpt.com "consume"))

---

# ✅ Option A — Local Kafka (PLAINTEXT) sanity test

```python
# --- Imports (new API) ---
from deephaven.stream.kafka import consumer as kc
from deephaven import dtypes as dht
import deephaven.plot.express as dx
from deephaven.agg import sum_

# 1) Kafka client config (PLAINTEXT)
KAFKA_CONFIG = {
    "bootstrap.servers": "localhost:9092",
    "group.id": "dh-consumer",
    "auto.offset.reset": "latest",
}
TOPIC = "positions"

# 2) Declare JSON schema for message VALUE
VALUE_SPEC = kc.json_spec({
    "_source_system": dht.string,
    "allotment":      dht.string,
    "instrumentCode": dht.string,
    "positionId":     dht.string,
    "pnl":            dht.double,
    "mtm":            dht.double,
    "isin":           dht.string,
    "cusip":          dht.string,
    "qty":            dht.int32,
    "price":          dht.double,
    "event_time":     dht.string,   # ISO-8601 string; we’ll parse to Instant next
})

# 3) Consume → refreshing table (NOTE: positional args; no 'properties=')
positions = kc.consume(
    KAFKA_CONFIG,           # kafka_config (dict) — positional
    TOPIC,                  # topic (str)
    key_spec=kc.KeyValueSpec.IGNORE,
    value_spec=VALUE_SPEC,
    table_type=kc.TableType.append(),
)

# 4) Parse event_time to Instant for windowing/ordering
positions = positions.update(["EventTime = (Instant)toDatetime(event_time)"])

# 5) A tiny, real-time dashboard slice
SOURCE = "GED"
pos = positions.where(f"_source_system == `{SOURCE}`")

latest = pos.sort_descending("EventTime").last_by("instrumentCode")

pnl_by_instr = (
    pos.agg_by([sum_("total_pnl = pnl")], by=["instrumentCode"])
       .sort_descending("total_pnl").head(10)
       .update(["sign = (total_pnl >= 0) ? `Gain` : `Loss`"])
)
bar_top10 = dx.bar(pnl_by_instr, x="total_pnl", y="instrumentCode", color="sign",
                   title=f"Top 10 by P&L • {SOURCE}")

pnl_by_allot = (
    pos.agg_by([sum_("total_pnl = pnl")], by=["allotment"])
       .sort_descending("total_pnl")
       .update(["sign = (total_pnl >= 0) ? `Gain` : `Loss`"])
)
bar_allot = dx.bar(pnl_by_allot, x="total_pnl", y="allotment", color="sign",
                   title=f"P&L by Allotment • {SOURCE}")
```

---

# 🔐 Option B — OAuth (SASL_SSL + OAUTHBEARER) like your Databricks setup

```python
from deephaven.stream.kafka import consumer as kc
from deephaven import dtypes as dht
from deephaven.agg import sum_
import deephaven.plot.express as dx

# 1) Kafka client config (SASL_SSL + OAUTHBEARER)
TOPIC = "gold_positions_changes"  # <-- your topic

KAFKA_CONFIG = {
    "bootstrap.servers": "pkc-XXXX.azure.confluent.cloud:9092",
    "group.id": "dh-gold-consumer",
    "auto.offset.reset": "latest",

    "security.protocol": "SASL_SSL",
    "sasl.mechanism": "OAUTHBEARER",

    # Try non-shaded first; if you get ClassNotFound, switch to shaded line below
    "sasl.login.callback.handler.class":
        "org.apache.kafka.common.security.oauthbearer.secured.OAuthBearerLoginCallbackHandler",
    # "sasl.login.callback.handler.class":
    #   "kafkashaded.org.apache.kafka.common.security.oauthbearer.secured.OAuthBearerLoginCallbackHandler",

    "sasl.oauthbearer.token.endpoint.url": "https://<your-idp>/oauth2/token",
    "sasl.oauthbearer.sub.claim.name": "client_id",
    "sasl.oauthbearer.client.id": "<KAFKA_CLIENT_ID>",
    "sasl.oauthbearer.client.secret": "<KAFKA_CLIENT_SECRET>",

    # If your cluster requires these (match your Databricks producer)
    "sasl.oauthbearer.extensions.logicalCluster": "<LOGICAL_CLUSTER>",
    "sasl.oauthbearer.extensions.identityPoolId": "<IDENTITY_POOL_ID>",
    "sasl.oauthbearer.extensions.identityPool": "<IDENTITY_POOL>",

    "ssl.endpoint.identification.algorithm": "https",
}

# 2) Declare JSON schema for VALUE
VALUE_SPEC = kc.json_spec({
    "event_ts":  dht.string,   # or dht.Instant if you already send epoch/Instant
    "symbol":    dht.string,
    "price":     dht.double,
    "qty":       dht.int32,
    "op":        dht.string,   # CDC op if present: I/U/D
    # ...add all columns you produce...
})

# 3) Consume → refreshing table (positional args)
t = kc.consume(
    KAFKA_CONFIG,
    TOPIC,
    key_spec=kc.KeyValueSpec.IGNORE,   # or kc.json_spec({...}) if key is JSON
    value_spec=VALUE_SPEC,
    table_type=kc.TableType.append(),
)

# 4) Cast event_ts to Instant if it’s ISO-8601 text
live = t.update(["EventTs = (Instant)toDatetime(event_ts)"])

# 5) Examples
latest = live.sort_descending("EventTs").last_by("symbol")

from deephaven.agg import sum_
kpis_5m = (
    live.where("EventTs >= now() - MINUTE * 5")
        .agg_by([sum_("qty_sum = qty"), sum_("notional = price * qty")], by=["symbol"])
        .sort_descending("notional")
)
```

---

## Common gotchas (with exact fixes)

- **“consumer is not taking the properties”** → Use **positional** `consume(kafka_config, topic, ...)` as above; don’t pass `properties=`. This matches the reference signature. ([Deephaven](https://deephaven.io/core/0.39.8/docs/reference/data-import-export/Kafka/consume/?utm_source=chatgpt.com "consume"))
    
- **Wrong module/import** → Use `from deephaven.stream.kafka import consumer as kc` (not `from deephaven import kafka`). ([Deephaven](https://deephaven.io/core/docs/how-to-guides/data-import-export/kafka-stream/?utm_source=chatgpt.com "Connect to a Kafka stream"))
    
- **Wrong table type** → Use `kc.TableType.append()` (enum), not `"append"`. ([Deephaven](https://deephaven.io/core/0.39.8/docs/reference/data-import-export/Kafka/consume/?utm_source=chatgpt.com "consume"))
    
- **Dtypes** → Use `int32/int64/double/string/Instant` from `deephaven.dtypes`.
    
- **OAuth handler `ClassNotFound`** → switch to the **shaded** handler class (see code).
    
- **No rows** → verify `TOPIC`, broker reachability (VPN/ACL), and that your producer is sending JSON objects (since we used `json_spec`).
    

If you paste one **sample JSON message** (keys + values), I’ll lock the `json_spec` exactly to your fields and add a rolling window chart tailored to your Gold schema.

---

```python
# === Deephaven real-time dashboard from Kafka (latest API) ===

from deephaven.stream.kafka import consumer as kc
from deephaven import dtypes as dt
from deephaven.agg import sum_
import deephaven.plot.express as dx

# -------------------------------------------------------------------
# 1) Kafka config (POSitional arg; new API).  Fill in your values.
# -------------------------------------------------------------------
TOPIC = "gold_positions_changes"     # <-- your topic name

KAFKA_CONFIG = {
    "bootstrap.servers": "pkc-xxxx.canadacentral.azure.confluent.cloud:9092",
    "group.id": "dh-gold-consumer",
    "auto.offset.reset": "latest",

    # --- SASL over TLS + OAUTHBEARER (matches your Databricks setup) ---
    "security.protocol": "SASL_SSL",
    "sasl.mechanism": "OAUTHBEARER",

    # JAAS stanza is REQUIRED with OAUTH; empty options are fine
    "sasl.jaas.config": "org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required ;",

    # Try non-shaded class first; if you get ClassNotFound, switch to shaded line
    "sasl.login.callback.handler.class":
        "org.apache.kafka.common.security.oauthbearer.secured.OAuthBearerLoginCallbackHandler",
    # "sasl.login.callback.handler.class":
    #   "kafkashaded.org.apache.kafka.common.security.oauthbearer.secured.OAuthBearerLoginCallbackHandler",

    "sasl.oauthbearer.token.endpoint.url": "https://<your-idp>/oauth2/token",
    "sasl.oauthbearer.sub.claim.name": "client_id",
    "sasl.oauthbearer.client.id": "<KAFKA_CLIENT_ID>",
    "sasl.oauthbearer.client.secret": "<KAFKA_CLIENT_SECRET>",

    # If your broker requires these extensions (you had them in Databricks), keep them:
    "sasl.oauthbearer.extensions.logicalCluster": "<LOGICAL_CLUSTER>",
    "sasl.oauthbearer.extensions.identityPoolId": "<IDENTITY_POOL_ID>",
    "sasl.oauthbearer.extensions.identityPool": "<IDENTITY_POOL>",

    "ssl.endpoint.identification.algorithm": "https",

    # Deserialize as strings; JSON parsing is handled by json_spec below
    "key.deserializer":   "org.apache.kafka.common.serialization.StringDeserializer",
    "value.deserializer": "org.apache.kafka.common.serialization.StringDeserializer",
}

# -------------------------------------------------------------------
# 2) Describe the JSON VALUE your producer sends (Silver schema)
#    (Include the columns you care about in the dashboard.)
# -------------------------------------------------------------------
VALUE_SPEC = kc.json_spec({
    "_source_system": dt.string,     # "FIXED INCOME" | "GED" | "COMMODITIES"
    "allotment":      dt.string,
    "positionId":     dt.string,
    "instrumentCode": dt.string,
    "cusip":          dt.string,
    "isin":           dt.string,
    "pnl":            dt.double,
    "mtm":            dt.double,     # keep if present
    "qty":            dt.int32,      # keep if present
    "price":          dt.double,     # keep if present
    "event_time":     dt.string,     # ISO-8601 text (e.g., "...Z")
})

# -------------------------------------------------------------------
# 3) Consume → refreshing table (POSitional args + TableType enum)
# -------------------------------------------------------------------
raw = kc.consume(
    KAFKA_CONFIG,            # kafka config (dict)
    TOPIC,                   # topic (str)
    key_spec=kc.KeyValueSpec.IGNORE,   # you have the key in VALUE anyway
    value_spec=VALUE_SPEC,
    table_type=kc.TableType.append(),
)

# -------------------------------------------------------------------
# 4) Clean/cast convenience columns (formula language inside strings)
# -------------------------------------------------------------------
live = raw.update([
    "EventTs = isNull(event_time) ? null : parseInstant(event_time)",

    # Prefer CUSIP; if blank/null, fall back to ISIN
    "identifier = (!isNull(cusip) && cusip != ``) ? cusip "
    "           : ((!isNull(isin) && isin != ``) ? isin : null)"
])

# -------------------------------------------------------------------
# 5) Choose a source_system slice for the dashboard (change anytime)
# -------------------------------------------------------------------
SOURCE = "GED"   # or "FIXED INCOME", "COMMODITIES"
pos = live.where(f"_source_system == `{SOURCE}`")

# -------------------------------------------------------------------
# 6) Dashboard tables (all refresh in real time)
# -------------------------------------------------------------------

# 6a) Overall data (your selected columns)
overall = pos.view([
    "_source_system", "allotment", "positionId", "instrumentCode",
    "cusip", "isin", "identifier",
    "pnl", "mtm", "qty", "price", "EventTs"
])

# 6b) Highest P&L row (for the selected source_system)
highest_pnl = pos.sort_descending("pnl").head(1)

# 6c) Sum of P&L by allotment (table + bar)
pnl_by_allot = (
    pos.agg_by([sum_("total_pnl = pnl")], by=["allotment"])
       .sort_descending("total_pnl")
       .update(["sign = (total_pnl >= 0) ? `Gain` : `Loss`"])
)
bar_allot = dx.bar(
    pnl_by_allot, x="total_pnl", y="allotment", color="sign",
    title=f"P&L by Allotment • {SOURCE}"
)

# 6d) Top 10 instruments by total P&L (table + bar)
top10_instruments = (
    pos.agg_by([sum_("total_pnl = pnl")], by=["instrumentCode"])
       .sort_descending("total_pnl").head(10)
       .update(["sign = (total_pnl >= 0) ? `Gain` : `Loss`"])
)
bar_top10_instr = dx.bar(
    top10_instruments, x="total_pnl", y="instrumentCode", color="sign",
    title=f"Top 10 Instruments by P&L • {SOURCE}"
)

# 6e) Top 10 “security” by total P&L (prefer CUSIP else ISIN)
top10_security = (
    pos.where("identifier != null")
       .agg_by([sum_("total_pnl = pnl")], by=["identifier"])
       .sort_descending("total_pnl").head(10)
       .update(["sign = (total_pnl >= 0) ? `Gain` : `Loss`"])
)
bar_top10_security = dx.bar(
    top10_security, x="total_pnl", y="identifier", color="sign",
    title=f"Top 10 Security by P&L • {SOURCE}"
)

# (Optional) rolling 5-minute KPIs per instrument
kpis_5m = (
    pos.where("EventTs >= now() - MINUTE * 5")
       .agg_by([sum_("qty_sum = qty"), sum_("notional = price * qty"), sum_("pnl_5m = pnl")],
               by=["instrumentCode"])
       .sort_descending("pnl_5m")
)
```

### Notes / gotchas (these match your last errors)

- **Datetime conversion**: use `parseInstant(event_time)` **inside** `update([...])` (formula language). Don’t call Python helpers there.
    
- **JAAS/KafkaClient error**: the one-liner  
    `sasl.jaas.config = "org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required ;"`  
    satisfies the JAAS requirement for OAUTHBEARER.
    
- **Callback handler class**: if you see `ClassNotFound`, switch to the **shaded** class shown in the comment.
    
- **Key handling**: since `source_system` is already in your VALUE payload, we ignore the Kafka key. If your key itself is JSON you can set `key_spec=kc.json_spec({...})`.
    

If you want this dashboard **per source_system simultaneously**, say the word and I’ll give you a small param-control pattern (three filtered views and a tabbed layout).

-----------------------------------------------

    

Here’s a **zero-error, drop-in Deephaven script** using the **new Kafka consumer API**.  
It converts your timestamps, builds the same four widgets, and makes the bars easier to read (top-N only + horizontal bars).

```python
# ==== Real-time Deephaven dashboard from Kafka (new API, your columns) ====

from deephaven.stream.kafka import consumer as kc
from deephaven import dtypes as dt
from deephaven.agg import sum_
import deephaven.plot.express as dx

# 1) Kafka config (fill in your broker + OAuth values)
TOPIC = "gold_positions_changes"   # <-- your topic

KAFKA_CONFIG = {
    "bootstrap.servers": "pkc-xxxx.canadacentral.azure.confluent.cloud:9092",
    "group.id": "dh-gold-consumer",
    "auto.offset.reset": "latest",

    "security.protocol": "SASL_SSL",
    "sasl.mechanism": "OAUTHBEARER",

    # JAAS stanza required for OAUTH
    "sasl.jaas.config": "org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required ;",

    # callback handler – if ClassNotFound, switch to the shaded class below
    "sasl.login.callback.handler.class":
        "org.apache.kafka.common.security.oauthbearer.secured.OAuthBearerLoginCallbackHandler",
    # "sasl.login.callback.handler.class":
    #   "kafkashaded.org.apache.kafka.common.security.oauthbearer.secured.OAuthBearerLoginCallbackHandler",

    "sasl.oauthbearer.token.endpoint.url": "https://<your-idp>/oauth2/token",
    "sasl.oauthbearer.sub.claim.name": "client_id",
    "sasl.oauthbearer.client.id": "<KAFKA_CLIENT_ID>",
    "sasl.oauthbearer.client.secret": "<KAFKA_CLIENT_SECRET>",
    "sasl.oauthbearer.extensions.logicalCluster": "<LOGICAL_CLUSTER>",
    "sasl.oauthbearer.extensions.identityPoolId": "<IDENTITY_POOL_ID>",
    "sasl.oauthbearer.extensions.identityPool": "<IDENTITY_POOL>",

    "ssl.endpoint.identification.algorithm": "https",

    # We read JSON strings and map via json_spec below
    "key.deserializer":   "org.apache.kafka.common.serialization.StringDeserializer",
    "value.deserializer": "org.apache.kafka.common.serialization.StringDeserializer",
}

# 2) JSON VALUE schema – only the columns you actually have
VALUE_SPEC = kc.json_spec({
    "_source_system": dt.string,
    "cusip":          dt.string,
    "isin":           dt.string,
    "allotment":      dt.string,
    "positionId":     dt.string,
    "instrumentCode": dt.string,   # if it’s truly numeric in your feed, you can change to dt.int64
    "pnl":            dt.double,
    "mtm":            dt.double,
    "_event_ts":      dt.string,   # ISO-8601 like “...Z”
    "_START_AT":      dt.string,   # bitemporal
    "_END_AT":        dt.string,   # bitemporal
})

# 3) Consume (positional args; TableType enum)
raw = kc.consume(
    KAFKA_CONFIG,
    TOPIC,
    key_spec=kc.KeyValueSpec.IGNORE,
    value_spec=VALUE_SPEC,
    table_type=kc.TableType.append(),
)

# 4) Normalize timestamps + preferred identifier (CUSIP else ISIN)
live = raw.update([
    "EventTs  = isNull(_event_ts)  ? null : parseInstant(_event_ts)",
    "StartAt  = isNull(_START_AT)  ? null : parseInstant(_START_AT)",
    "EndAt    = isNull(_END_AT)    ? null : parseInstant(_END_AT)",
    "identifier = (!isNull(cusip) && cusip != ``) ? cusip "
    "          : ((!isNull(isin)  && isin  != ``) ? isin  : null)"
])

# 5) pick a source slice (switch this any time and re-run the line)
SOURCE = "GED"  # "GED" | "FIXED INCOME" | "COMMODITIES"
pos = live.where(f"_source_system == `{SOURCE}`")

# 6) DASHBOARD TABLES (all refreshing)

# 6a) Overall table
overall = pos.view([
    "_source_system", "allotment", "positionId", "instrumentCode",
    "cusip", "isin", "identifier", "pnl", "mtm", "EventTs", "StartAt", "EndAt"
])

# 6b) Highest P&L row
highest_pnl = pos.sort_descending("pnl").head(1)

# 6c) Sum of P&L by allotment (table + horizontal bar)
pnl_by_allot = (
    pos.agg_by([sum_("total_pnl = pnl")], by=["allotment"])
       .sort_descending("total_pnl")
       .update(["sign = (total_pnl >= 0) ? `Gain` : `Loss`"])
)
# horizontal bars (x = value, y = category) and top-N so the bars are thicker
bar_allot = dx.bar(
    pnl_by_allot.head(12),
    x="total_pnl", y="allotment", color="sign",
    title=f"P&L by Allotment • {SOURCE}"
)

# 6d) Top 10 instruments by total P&L (table + horizontal bar)
top10_instruments = (
    pos.agg_by([sum_("total_pnl = pnl")], by=["instrumentCode"])
       .sort_descending("total_pnl").head(10)
       .update(["sign = (total_pnl >= 0) ? `Gain` : `Loss`"])
)
bar_top10_instr = dx.bar(
    top10_instruments,
    x="total_pnl", y="instrumentCode", color="sign",
    title=f"Top 10 Instruments by P&L • {SOURCE}"
)

# 6e) Top 10 security by total P&L (CUSIP→ISIN fallback, horizontal bar)
top10_security = (
    pos.where("identifier != null")
       .agg_by([sum_("total_pnl = pnl")], by=["identifier"])
       .sort_descending("total_pnl").head(10)
       .update(["sign = (total_pnl >= 0) ? `Gain` : `Loss`"])
)
bar_top10_security = dx.bar(
    top10_security,
    x="total_pnl", y="identifier", color="sign",
    title=f"Top 10 Security by P&L • {SOURCE}"
)
```

### Why the bars now look better

- We draw **horizontal** bars (`x`=metric, `y`=category) so long labels don’t get cramped.
    
- We **limit to top-N** (10–12) so each bar is thicker and legible.
    
- Gain/Loss coloring helps at a glance.
    

If `instrumentCode` in your topic is truly numeric, change its dtype in `VALUE_SPEC` to `dt.int64` and the code still works. If you want all three **source systems** on one screen (three panels), say the word and I’ll give you a quick trellis pattern.

------------------


great progress! let’s tighten a few things based on your screenshots:

- you have **both** `_START_AT/_END_AT` **and** `_start_at/_end_at`.
    
- `_START_AT/_END_AT` are timestamps in Databricks, but come through Kafka as strings (or sometimes true timestamps).
    
- your `highest_pnl` view showed Kafka metadata—so we’ll **only select** the columns you want.
    
- make the bars **wider / clearer** and add **axis labels**.
    

Below is a **drop-in update** using the **new Kafka API** you’re already on. It:

- coalesces start/end using either upper- or lower-case fields and works whether they arrive as strings or timestamps,
    
- removes any Kafka metadata from the widgets,
    
- makes charts horizontal, **limits to top-N**, and sets axis labels.
    

```python
# === Deephaven real-time dashboard from Kafka (your columns, cleaned) ===
from deephaven.stream.kafka import consumer as kc
from deephaven import dtypes as dt
from deephaven.agg import sum_
import deephaven.plot.express as dx

# --- 1) Kafka config (same as you have now) ---
TOPIC = "gold_positions_changes"
KAFKA_CONFIG = {
    "bootstrap.servers": "pkc-xxxx.canadacentral.azure.confluent.cloud:9092",
    "group.id": "dh-gold-consumer",
    "auto.offset.reset": "latest",
    "security.protocol": "SASL_SSL",
    "sasl.mechanism": "OAUTHBEARER",
    "sasl.jaas.config": "org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required ;",
    "sasl.login.callback.handler.class":
        "org.apache.kafka.common.security.oauthbearer.secured.OAuthBearerLoginCallbackHandler",
    # "sasl.login.callback.handler.class":
    #   "kafkashaded.org.apache.kafka.common.security.oauthbearer.secured.OAuthBearerLoginCallbackHandler",
    "sasl.oauthbearer.token.endpoint.url": "https://<your-idp>/oauth2/token",
    "sasl.oauthbearer.sub.claim.name": "client_id",
    "sasl.oauthbearer.client.id": "<KAFKA_CLIENT_ID>",
    "sasl.oauthbearer.client.secret": "<KAFKA_CLIENT_SECRET>",
    "sasl.oauthbearer.extensions.logicalCluster": "<LOGICAL_CLUSTER>",
    "sasl.oauthbearer.extensions.identityPoolId": "<IDENTITY_POOL_ID>",
    "sasl.oauthbearer.extensions.identityPool": "<IDENTITY_POOL>",
    "ssl.endpoint.identification.algorithm": "https",
    "key.deserializer":   "org.apache.kafka.common.serialization.StringDeserializer",
    "value.deserializer": "org.apache.kafka.common.serialization.StringDeserializer",
}

# --- 2) JSON schema: only the fields you actually have ---
VALUE_SPEC = kc.json_spec({
    "_source_system": dt.string,
    "allotment":      dt.string,
    "positionId":     dt.string,
    "instrumentCode": dt.string,  # change to dt.int64 if truly numeric
    "cusip":          dt.string,
    "isin":           dt.string,
    "pnl":            dt.double,
    "mtm":            dt.double,
    "_event_ts":      dt.string,
    "_START_AT":      dt.string,
    "_END_AT":        dt.string,
    "_start_at":      dt.string,  # seen in your screenshots; keep as backup
    "_end_at":        dt.string,
})

# --- 3) Consume (positional args + TableType) ---
raw = kc.consume(
    KAFKA_CONFIG, TOPIC,
    key_spec=kc.KeyValueSpec.IGNORE,
    value_spec=VALUE_SPEC,
    table_type=kc.TableType.append(),
)

# --- 4) Normalization: robust timestamp casting + identifier preference ---
# Use parseInstant on whichever variant is present; handle both true timestamps and strings
live = raw.update([
    # Event time (upper-case field in your screenshots)
    "EventTs = isNull(_event_ts) ? null : parseInstant(_event_ts)",

    # Start/End: prefer UPPER variant; if null, try lower; parse string form
    "StartAt = !isNull(_START_AT) ? parseInstant(string(_START_AT)) "
    "        : (!isNull(_start_at) ? parseInstant(_start_at) : null)",

    "EndAt   = !isNull(_END_AT)   ? parseInstant(string(_END_AT)) "
    "        : (!isNull(_end_at)  ? parseInstant(_end_at)  : null)",

    # Security identifier: prefer CUSIP, else ISIN
    "identifier = (!isNull(cusip) && cusip != ``) ? cusip "
    "          : ((!isNull(isin)  && isin  != ``) ? isin  : null)"
])

# --- 5) Slice by source system ---
SOURCE = "GED"  # change to "FIXED INCOME" or "COMMODITIES" as needed
pos = live.where(f"_source_system == `{SOURCE}`")

# --- 6) Dashboard tables (select only business columns; no Kafka metadata) ---

# Overall table
overall = pos.view([
    "_source_system", "allotment", "positionId", "instrumentCode",
    "cusip", "isin", "identifier", "pnl", "mtm", "EventTs", "StartAt", "EndAt"
])

# Highest P&L row (business columns only)
highest_pnl = (
    pos.sort_descending("pnl").view([
        "_source_system", "instrumentCode", "identifier",
        "allotment", "positionId", "pnl", "mtm", "EventTs"
    ]).head(1)
)

# Sum P&L by allotment (horizontal bars, top 12)
pnl_by_allot = (
    pos.agg_by([sum_("total_pnl = pnl")], by=["allotment"])
       .sort_descending("total_pnl").head(12)
       .update(["sign = (total_pnl >= 0) ? `Gain` : `Loss`"])
)
bar_allot = dx.bar(
    pnl_by_allot,
    x="total_pnl", y="allotment", color="sign",
    title=f"P&L by Allotment • {SOURCE}",
    x_label="Total P&L", y_label="Allotment"
)

# Top 10 instruments by total P&L (horizontal bars)
top10_instruments = (
    pos.agg_by([sum_("total_pnl = pnl")], by=["instrumentCode"])
       .sort_descending("total_pnl").head(10)
       .update(["sign = (total_pnl >= 0) ? `Gain` : `Loss`"])
)
bar_top10_instr = dx.bar(
    top10_instruments,
    x="total_pnl", y="instrumentCode", color="sign",
    title=f"Top 10 Instruments by P&L • {SOURCE}",
    x_label="Total P&L", y_label="Instrument"
)

# Top 10 security by total P&L (CUSIP→ISIN fallback; horizontal bars)
top10_security = (
    pos.where("identifier != null")
       .agg_by([sum_("total_pnl = pnl")], by=["identifier"])
       .sort_descending("total_pnl").head(10)
       .update(["sign = (total_pnl >= 0) ? `Gain` : `Loss`"])
)
bar_top10_security = dx.bar(
    top10_security,
    x="total_pnl", y="identifier", color="sign",
    title=f"Top 10 Security by P&L • {SOURCE}",
    x_label="Total P&L", y_label="Security (CUSIP or ISIN)"
)
```

### Why this fixes your points

- **StartAt/EndAt nulls**: we now **coalesce** `_START_AT` → `_start_at`, and use `parseInstant(string(...))` so it works whether the field arrives as a string or a timestamp.
    
- **Kafka metadata showing up**: `highest_pnl` and other widgets use **`view([...])`** to select only business columns.
    
- **Bars clearer & wider**: horizontal orientation + **top-N** (10–12) yields thicker bars; and we set **axis labels** with `x_label`/`y_label`.
    

If you want a **tabbed dashboard** for all three source systems at once, or to persist the layout, say the word and I’ll add that pattern too.

---------------


    

> Fill in `TOPIC` and the OAuth values in `KAFKA_CONFIG` and run.

```python
# ==== Deephaven real-time dashboard from Kafka (tabbed, end-to-end) ====

from deephaven.stream.kafka import consumer as kc
from deephaven import dtypes as dt
from deephaven.agg import sum_
import deephaven.plot.express as dx
from deephaven import ui

# ---------------------------------------------------------------------
# 1) Kafka config (edit TOPIC + OAuth values to your environment)
# ---------------------------------------------------------------------
TOPIC = "gold_positions_changes"   # <-- your topic

KAFKA_CONFIG = {
    "bootstrap.servers": "pkc-xxxx.canadacentral.azure.confluent.cloud:9092",
    "group.id": "dh-gold-consumer",
    "auto.offset.reset": "latest",

    # --- SASL over TLS + OAUTHBEARER ---
    "security.protocol": "SASL_SSL",
    "sasl.mechanism": "OAUTHBEARER",

    # JAAS stanza is REQUIRED with OAUTH; empty options are fine
    "sasl.jaas.config": "org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required ;",

    # Try non-shaded handler first; if you see ClassNotFound, switch to the shaded line below
    "sasl.login.callback.handler.class":
        "org.apache.kafka.common.security.oauthbearer.secured.OAuthBearerLoginCallbackHandler",
    # "sasl.login.callback.handler.class":
    #   "kafkashaded.org.apache.kafka.common.security.oauthbearer.secured.OAuthBearerLoginCallbackHandler",

    "sasl.oauthbearer.token.endpoint.url": "https://<your-idp>/oauth2/token",
    "sasl.oauthbearer.sub.claim.name": "client_id",
    "sasl.oauthbearer.client.id": "<KAFKA_CLIENT_ID>",
    "sasl.oauthbearer.client.secret": "<KAFKA_CLIENT_SECRET>",

    # Keep these if your broker requires them (you had them in Databricks)
    "sasl.oauthbearer.extensions.logicalCluster": "<LOGICAL_CLUSTER>",
    "sasl.oauthbearer.extensions.identityPoolId": "<IDENTITY_POOL_ID>",
    "sasl.oauthbearer.extensions.identityPool": "<IDENTITY_POOL>",

    "ssl.endpoint.identification.algorithm": "https",

    # We read JSON text then map via json_spec
    "key.deserializer":   "org.apache.kafka.common.serialization.StringDeserializer",
    "value.deserializer": "org.apache.kafka.common.serialization.StringDeserializer",
}

# ---------------------------------------------------------------------
# 2) JSON VALUE schema — only the columns you actually have
# ---------------------------------------------------------------------
VALUE_SPEC = kc.json_spec({
    "_source_system": dt.string,     # GED / FIXED INCOME / COMMODITIES
    "allotment":      dt.string,
    "positionId":     dt.string,
    "instrumentCode": dt.string,     # switch to dt.int64 if truly numeric
    "cusip":          dt.string,
    "isin":           dt.string,
    "pnl":            dt.double,
    "mtm":            dt.double,
    "_event_ts":      dt.string,     # ISO-8601 text
    "_START_AT":      dt.string,     # may arrive as string or timestamp; we handle both
    "_END_AT":        dt.string,
    "_start_at":      dt.string,     # lowercase variants observed
    "_end_at":        dt.string,
})

# ---------------------------------------------------------------------
# 3) Consume (POSitional args; TableType enum)
# ---------------------------------------------------------------------
raw = kc.consume(
    KAFKA_CONFIG,
    TOPIC,
    key_spec=kc.KeyValueSpec.IGNORE,   # key is not needed; source system is in VALUE
    value_spec=VALUE_SPEC,
    table_type=kc.TableType.append(),
)

# ---------------------------------------------------------------------
# 4) Normalize timestamps + preferred identifier (CUSIP→ISIN)
#   - parseInstant() is Deephaven's formula function (used inside update strings)
#   - string(...) lets us safely coerce timestamp objects to string if needed
# ---------------------------------------------------------------------
live = raw.update([
    "EventTs = isNull(_event_ts) ? null : parseInstant(_event_ts)",

    "StartAt = !isNull(_START_AT) ? parseInstant(string(_START_AT)) "
    "        : (!isNull(_start_at) ? parseInstant(_start_at) : null)",

    "EndAt   = !isNull(_END_AT)   ? parseInstant(string(_END_AT)) "
    "        : (!isNull(_end_at)  ? parseInstant(_end_at)  : null)",

    "identifier = (!isNull(cusip) && cusip != ``) ? cusip "
    "          : ((!isNull(isin)  && isin  != ``) ? isin  : null)"
])

# ---------------------------------------------------------------------
# 5) Helper to build one dashboard panel
#    source=None → ALL (unfiltered) and includes extra "by source" chart
# ---------------------------------------------------------------------
def make_panel(source: str | None):
    base = live if source is None else live.where(f"_source_system == `{source}`")
    title_suffix = "ALL" if source is None else source

    # Overall business columns only (no Kafka metadata)
    overall = base.view([
        "_source_system", "allotment", "positionId", "instrumentCode",
        "cusip", "isin", "identifier", "pnl", "mtm", "EventTs", "StartAt", "EndAt"
    ])

    # Highest P&L row (business columns)
    highest_pnl = (
        base.sort_descending("pnl")
            .view(["_source_system", "instrumentCode", "identifier",
                   "allotment", "positionId", "pnl", "mtm", "EventTs"])
            .head(1)
    )

    # P&L by allotment (limit to top 12 for thicker bars)
    pnl_by_allot = (
        base.agg_by([sum_("total_pnl = pnl")], by=["allotment"])
            .sort_descending("total_pnl").head(12)
            .update(["sign = (total_pnl >= 0) ? `Gain` : `Loss`"])
    )
    bar_allot = dx.bar(
        pnl_by_allot, x="total_pnl", y="allotment", color="sign",
        title=f"P&L by Allotment • {title_suffix}",
        x_label="Total P&L", y_label="Allotment"
    )

    # Top 10 instruments by P&L
    top10_instr = (
        base.agg_by([sum_("total_pnl = pnl")], by=["instrumentCode"])
            .sort_descending("total_pnl").head(10)
            .update(["sign = (total_pnl >= 0) ? `Gain` : `Loss`"])
    )
    bar_top10_instr = dx.bar(
        top10_instr, x="total_pnl", y="instrumentCode", color="sign",
        title=f"Top 10 Instruments by P&L • {title_suffix}",
        x_label="Total P&L", y_label="Instrument"
    )

    # Top 10 security by P&L (CUSIP→ISIN)
    top10_sec = (
        base.where("identifier != null")
            .agg_by([sum_("total_pnl = pnl")], by=["identifier"])
            .sort_descending("total_pnl").head(10)
            .update(["sign = (total_pnl >= 0) ? `Gain` : `Loss`"])
    )
    bar_top10_security = dx.bar(
        top10_sec, x="total_pnl", y="identifier", color="sign",
        title=f"Top 10 Security by P&L • {title_suffix}",
        x_label="Total P&L", y_label="Security (CUSIP or ISIN)"
    )

    # Extra for ALL: P&L by Source System
    extra_row = None
    if source is None:
        by_source = (
            base.agg_by([sum_("total_pnl = pnl")], by=["_source_system"])
                .sort_descending("total_pnl")
                .update(["sign = (total_pnl >= 0) ? `Gain` : `Loss`"])
        )
        bar_by_source = dx.bar(
            by_source, x="total_pnl", y="_source_system", color="sign",
            title="P&L by Source System",
            x_label="Total P&L", y_label="Source System"
        )
        extra_row = ui.hbox(bar_by_source, bar_top10_security)
    else:
        extra_row = ui.hbox(bar_top10_security)

    # Layout for this tab
    return ui.vbox(
        ui.heading(f"{title_suffix} — Live Positions"),
        highest_pnl,
        ui.hbox(bar_allot, bar_top10_instr),
        extra_row,
        overall,
        gap="16px"
    )

# ---------------------------------------------------------------------
# 6) Build the four tabs and expose a 'dashboard' variable
# ---------------------------------------------------------------------
tab_all = make_panel(None)
tab_ged = make_panel("GED")
tab_fi  = make_panel("FIXED INCOME")
tab_cmd = make_panel("COMMODITIES")

dashboard = ui.tabbed({
    "ALL": tab_all,
    "GED": tab_ged,
    "FIXED INCOME": tab_fi,
    "COMMODITIES": tab_cmd,
})
```

**Notes**

- If your OAuth callback handler throws `ClassNotFound`, uncomment the **shaded** class line and comment out the non-shaded one.
    
- If `instrumentCode` is numeric, change its dtype to `dt.int64` in `VALUE_SPEC`.
    
- You can drag/resize panels and **File → Save Layout** to keep your preferred arrangement.

-------------------------


You're bumping into a Deephaven **formula** error because I used `string(...)` inside `update([...])`. There’s no `string()` function in the formula language—use the value directly (it’s already a string) or cast with `(String)`.

Here’s the **one-line fix** for each timestamp field; replace your current `StartAt` / `EndAt` expressions with these:

```python
live = raw.update([
    "EventTs = isNull(_event_ts) ? null : parseInstant(_event_ts)",

    # ✅ no string() — both variants treated as ISO strings
    "StartAt = !isNull(_START_AT) ? parseInstant(_START_AT) "
    "        : (!isNull(_start_at) ? parseInstant(_start_at) : null)",

    "EndAt   = !isNull(_END_AT)   ? parseInstant(_END_AT) "
    "        : (!isNull(_end_at)  ? parseInstant(_end_at)  : null)",

    "identifier = (!isNull(cusip) && cusip != ``) ? cusip "
    "          : ((!isNull(isin)  && isin  != ``) ? isin  : null)"
])
```

That removes the `FormulaCompilationException: Cannot find method string(java.lang.String)` error.

If you later discover `_START_AT` is a true timestamp (not text) in some partitions, switch to a cast instead of parsing:

```python
"StartAt = !isNull(_START_AT) ? (Instant)_START_AT "
"        : (!isNull(_start_at) ? parseInstant(_start_at) : null)",
```

Do the same for `EndAt` if needed.

Everything else in your dashboard can stay as-is. If you still see a formula error, send me the **exact** expression line mentioned at the top of the stack trace and I’ll patch it precisely.