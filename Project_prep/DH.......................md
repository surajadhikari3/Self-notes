You got it—here’s the **minimal, copy-paste recipe** to read **Kafka → Deephaven (local)** and render your **real-time dashboard**. No Docker needed.

---

# 0) One-time setup (local)

**Deephaven (no Docker)**

```bash
python -m venv dh-env
# Win: dh-env\Scripts\activate    Mac/Linux: source dh-env/bin/activate
python -m pip install --upgrade pip
python -m pip install deephaven-server
```

Create `start_dh.py`:

```python
from deephaven_server import Server
s = Server(port=10000, jvm_args=["-Xmx4g"])
s.start()
print("Deephaven IDE → http://localhost:10000/ide/  (Ctrl+C to stop)")
import time;  [time.sleep(60) for _ in iter(int,1)]
```

Run: `python start_dh.py` and open **[http://localhost:10000/ide/](http://localhost:10000/ide/)**.

**Kafka (local KRaft)**

- Install Kafka and start a broker on `localhost:9092`.
    
- Create topic:
    

```bash
kafka-topics --bootstrap-server localhost:9092 --create --topic positions --partitions 3 --replication-factor 1
```

(Optional) **Test producer** (quick JSON generator):

```bash
pip install kafka-python
python - << "PY"
import json, time, uuid, random, string
from datetime import datetime, timezone
from kafka import KafkaProducer
SOURCES=["GED","FIXED INCOME","COMMODITIES"]
ALLOT=["GovernmentBond","CorporateBond","Futures","Options","CommodityFund","Equity","REIT"]
producer=KafkaProducer(bootstrap_servers="localhost:9092",
                       value_serializer=lambda v: json.dumps(v).encode())
def rand(n,alphabet): return "".join(random.choices(alphabet,k=n))
while True:
    now=datetime.now(timezone.utc).isoformat()
    for _ in range(60):
        src=random.choice(SOURCES)
        msg={
          "_source_system":src,
          "allotment":random.choice(ALLOT),
          "instrumentCode":f"{src.split()[0][:3].upper()}{random.randint(1,99999):05d}",
          "positionId":str(uuid.uuid4()),
          "pnl":round(random.gauss(0,300),2),
          "mtm":round(random.uniform(100,50000),2),
          "isin":rand(12,string.ascii_uppercase+"0123456789"),
          "cusip":rand(9,string.ascii_uppercase+"0123456789"),
          "qty":random.randint(1,300),
          "price":round(random.uniform(5,500),2),
          "event_time":now
        }
        producer.send("positions", value=msg)
    producer.flush(); time.sleep(1)
PY
```

---

# 1) Deephaven: read Kafka and build the 4-widget dashboard

In the Deephaven IDE, create a **Python** script and paste all of this:

```python
# === Kafka → Deephaven live dashboard ===
from deephaven import kafka, dtypes as dht
from deephaven.agg import sum_
import deephaven.plot.express as dx
from deephaven.time import to_datetime

# ---- 1) Define JSON schema of your Kafka message (value) ----
value_spec = kafka.json_spec([
    ("_source_system", dht.string),
    ("allotment",      dht.string),
    ("instrumentCode", dht.string),
    ("positionId",     dht.string),
    ("pnl",            dht.double),
    ("mtm",            dht.double),
    ("isin",           dht.string),
    ("cusip",          dht.string),
    ("qty",            dht.int_),
    ("price",          dht.double),
    ("event_time",     dht.string),  # ISO-8601
])

# ---- 2) Consume the topic as a LIVE append table (push-based) ----
positions = kafka.consume(
    {'bootstrap.servers': 'localhost:9092', 'auto.offset.reset': 'latest'},
    topic='positions',
    value_spec=value_spec,
    table_type='append'
)

# Optional: parse event_time to an Instant (useful for windowing)
positions = positions.update(["EventTime = toDatetime(event_time)"])

# ---- 3) Parameter: pick your _source_system (GED / FIXED INCOME / COMMODITIES) ----
SOURCE = "GED"

# Slice the live table once; all widgets reuse this
pos = positions.where(f"_source_system == `{SOURCE}`")

# ---- 4) Widget 1: Position Data (overall table) ----
overall = pos.view([
    "_source_system", "allotment", "instrumentCode", "positionId",
    "pnl", "mtm", "isin", "cusip", "qty", "price", "EventTime"
])

# ---- 5) Widget 2: Highest P&L (single row) ----
highest = pos.sort_descending("pnl").head(1)

# ---- 6) Widget 3: Sum of P&L by allotment (bar) ----
pnl_by_allot = (
    pos.agg_by([sum_("total_pnl = pnl")], by=["allotment"])
       .sort_descending("total_pnl")
       .update(["sign = (total_pnl >= 0) ? `Gain` : `Loss`"])
)
bar_allot = dx.bar(
    pnl_by_allot, x="total_pnl", y="allotment", color="sign",
    title=f"Sum of P&L by Allotment • {SOURCE}"
)

# ---- 7) Widget 4: Top-10 instruments by P&L (bar) ----
top10 = (
    pos.agg_by([sum_("total_pnl = pnl")], by=["instrumentCode"])
       .sort_descending("total_pnl").head(10)
       .update(["sign = (total_pnl >= 0) ? `Gain` : `Loss`"])
)
bar_top10 = dx.bar(
    top10, x="total_pnl", y="instrumentCode", color="sign",
    title=f"Top 10 Instruments by P&L • {SOURCE}"
)

# ---- 8) (Optional) KPIs table ----
kpis = pos.agg_by([sum_("total_pnl = pnl")], by=[])
```

### What you’ll see

- `positions` ticks as Kafka messages arrive (no polling).
    
- `overall`, `highest`, `pnl_by_allot`/`bar_allot`, `top10`/`bar_top10` all **update in real time**.
    
- Change `SOURCE = "FIXED INCOME"` or `"COMMODITIES"`, run that line again to switch the entire dashboard.
    

> Tip: arrange your tables & charts in the UI and **File → Save Layout**.

---

## Quick checks if something doesn’t tick

- In the Deephaven console, run `positions.size()` — it should increase over time.
    
- Ensure your producer is writing to **topic `positions`** on **localhost:9092**.
    
- If you changed message fields, update `value_spec` accordingly.
    
- If you see offset errors, stop/start the script (keeps consuming from `latest` by default).
    

---

That’s it—you’re reading Kafka in Deephaven locally and rendering the same four widgets, with **true push-based updates**.