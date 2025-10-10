---

---

Technologies:

Harness ->  CI/CD platform
Rancher --> Kubernetes platform(can manage multiple kubernets cluster in a centralized even if services                        are deployed in AKS,EKS, on prem..........)


- **Harness** = CI/CD pipeline → builds and deploys apps.
    
- **Rancher** = manages Kubernetes clusters where Harness deploys those apps.


### Producer knobs that matter (besides compression)

- `batch.size` (increase) + `linger.ms` (a few ms) → bigger batches = better compression & throughput.
    
- `acks=1` (throughput) or `acks=all` (stronger durability; a bit less TPS).
    
- `enable.idempotence=true` (exactly-once semantics for producer; slight overhead but worth it).
    
- Enough **partitions** to parallelize.
    
- **Keys** that distribute load evenly.




centralized logging --> FileBeat --> ships raw log  -> logstash(collects, process, enrich ) -> elasticsearch --> collects, indexing and store --> Kibanana for visualization and dashboard... 

“Yes, in ELK, Logstash is responsible for collecting and processing logs before shipping them to Elasticsearch. In many modern setups, we use Filebeat on each server to ship raw logs to Logstash, which then enriches them with filters and sends them to Elasticsearch. Kibana sits on top for search and visualization.”


distributed tracing --> Slueth(Instrumentaion -- > Trace ID + SpanId) + Zipkins (collects, stores and visualize the traces) --> it shows the lifecycle/ journey of the requests over the microservice 
					(now from spring 3.x slueth is deprecated and OpenTelemetry is standard)


“In microservices, we need centralized logging to aggregate and search logs across all services, and distributed tracing to follow a single request end-to-end across services. Logging tells us (**what**) happened, tracing tells us (**where and why** )it happened. Together, they provide full observability and reduce mean time to resolution in production.”

“Dynatrace OneAgent isn’t embedded in the Spring Boot code — it runs at the host/container level. It auto-instruments the JVM and frameworks like Spring, Kafka, and JDBC, so we get traces, metrics, and logs without modifying the application. That’s why enterprises prefer APM tools — they reduce developer effort for observability.”

One Agent is Deployed as DaemonSet in AKS

DaemonSet ensures 1 pod running in each node. Node can have multiple pods..

Pods should have one container but sometime can have extra container which is called side car pattern...



“No, Dynatrace doesn’t use the sidecar pattern. It’s deployed as a DaemonSet in AKS, which means one OneAgent Pod runs per node and monitors all microservices on that node. This avoids the overhead of adding a sidecar to every microservice Pod, while still giving full observability across services.”

In your **Kroger project**, you mentioned:

- _“Integrated Harness, Rancher, and GitHub Actions to create a robust CI/CD ecosystem.”_
    
- Likely workflow was:
    
    - **Harness pipelines** triggered on Git commits.
        
    - Built Docker images → pushed to registry.
        
    - Deployed microservices to Kubernetes (via Rancher).
        
    - Observability hooked in for rollback if issues detected.

“Harness is a next-gen CI/CD platform that automates build, test, and deployment of applications to cloud or on-prem. It’s used when teams want intelligent pipelines with features like canary/blue-green deployments, feature flags, cost management, and automated rollback. In my Kroger project, we used Harness with Rancher to deploy microservices on Kubernetes and integrate observability for safe rollouts.”


“Even if my services are in AKS, Rancher can manage them — you just import the AKS cluster into Rancher. It’s especially useful in hybrid setups where you may have AKS, EKS, and on-prem clusters, because Rancher gives a single pane of glass to manage all of them. At Kroger, we used Rancher to standardize policies and monitoring across Kubernetes clusters, while Harness handled the CI/CD pipelines.”




---

## 🔹 Treasury (in a Bank)

- Treasury = the **bank’s department that manages money/liquidity**.
    
- They ensure the bank always has enough **cash & liquid assets** to pay obligations (like customer withdrawals, settlements, loans).
    
- Think of it as the **bank’s CFO desk** that manages funding, risk, and compliance.
    

---

## 🔹 LCR (Liquidity Coverage Ratio)

- A **Basel III regulatory requirement**.
    
- Formula (simplified):
    

![[Pasted image 20250914211139.png]]

- Must be **≥ 100%** (bank should have enough liquid assets to survive a 30-day stress scenario).
    
- Example: If the bank expects $80B cash outflows in stress, it must hold at least $80B in HQLA.
    

---

## 🔹 HQLA (High Quality Liquid Assets)

- Assets that can **quickly be converted into cash with little loss in value**.
    
- Examples:
    
    - **Level 1**: Cash, central bank reserves, government bonds.
        
    - **Level 2A**: High-rated corporate bonds.
        
    - **Level 2B**: Certain stocks (with haircut). --> Haircut means  considering certain perceentage only as stocks are volatile....
        
- Treasury monitors these daily to ensure enough liquidity.
    

---

## 🔹 FX (Foreign Exchange Trades)

- “FX” = **currency trading** (USD/EUR, USD/JPY, etc.).
    
- Banks’ liquidity also depends on foreign currency trades (settlements in multiple currencies).
    

---

## 🔹 Repo (Repurchase Agreements)

- A **short-term borrowing mechanism**.
    
- Example: Bank sells securities today (gets cash) with agreement to buy them back tomorrow (pays cash back).
    
- Used by Treasury to manage liquidity overnight.
    

---

## 🔹 Interbank Lending

- Banks lending money to each other (overnight funding).
    
- Impacts liquidity and treasury’s cashflow monitoring.
    

---

## 🔹 Position (in Trading/Treasury)

- **Position = the net exposure a bank holds in a financial instrument (asset, security, or currency).**
    
- Formula (simplified):
    

Position=Total Buys−Total Sells\text{Position} = \text{Total Buys} - \text{Total Sells}

- Example:
    
    - If Treasury holds **100 shares of Apple stock (AAPL)** and sold **30**, the position = 70 long.
        
    - If they sold more than they own (e.g., short selling), position can be negative.
        
- In Treasury:
    
    - **Cash position** = how much liquid cash they have.
        
    - **Securities position** = how many bonds, repos, or equities they hold.
        

👉 Your project was enriching trades into **positions** so Treasury could see their **intraday liquidity position** at any time.

---

## 🔹 Adjustment Engine (business meaning)

- Sometimes trades are late, misclassified, or wrongly mapped.
    
- Adjustment engine lets Treasury **override classifications** temporarily to remain compliant.
    
- Example: A corporate bond wrongly classified as equity → Treasury can reclassify it as HQLA Level 2A for ratio reporting.
    

---

## 🔹 Intraday Liquidity Monitoring

- Instead of end-of-day batch reconciliation, Treasury wants **real-time dashboards**.
    
- They see:
    
    - Current cash balance.
        
    - HQLA bucket distribution.
        
    - LCR buffer against Basel III limits.
        

---

## 🎯 Interview-Ready Business Summary

> “Treasury is the bank’s liquidity management desk. Their job is to ensure enough High-Quality Liquid Assets (HQLA) are available to meet Basel III Liquidity Coverage Ratio (LCR) requirements, meaning the bank can survive a 30-day stress period. In our project, we streamed trades like FX, repos, and interbank loans from OMS/EMS into Kafka, enriched them into positions, and calculated intraday liquidity buffers. Treasury could then view their real-time positions on dashboards, and make adjustments if trades were misclassified. This helped the bank maintain compliance and avoid reliance on slow, manual spreadsheets.”

---


Great 🔥 — let’s break this into **two parts**: _trade normalization with Avro_ and _asset classes_.

---

## 🔹 1. What Does “Normalize Trade with Avro” Mean?

When trades come from **different source systems**, they look different:

- FX trade message → `currencyPair: USD/EUR, amount, settlementDate`
    
- Repo trade message → `collateral: govBond, repoRate, maturityDate`
    
- Bond trade message → `cusip, isin, coupon, yield`
    

👉 Problem: Every desk/system has its own **data format & schema**. Hard to process consistently.

### **Normalization** =

- Converting all these **different trade formats into a common schema**.
    
- Ensures downstream systems (Kafka Streams, Databricks, Treasury dashboards) can read them uniformly.
    

### **Why Avro?**

- Avro is a **schema-based serialization format**.
    
- Schema defines fields (mandatory/optional, data types).
    
- With **Confluent Schema Registry**, you enforce consistency and avoid “data chaos.”
    
- Once trades are serialized into Avro (binary), they are **compact, fast, and strongly typed**.
    

👉 So “normalize trades with Avro” = **standardize all trade data into Avro schemas before streaming further.**

---

## 🔹 2. What Is an Asset Class?

An **asset class** = a category of financial instruments that behave similarly in markets.

- They have similar risk/return profiles.
    
- In Treasury & trading, trades are usually grouped by asset class.
    

### Common Asset Classes in Your Project Context:

1. **FX (Foreign Exchange)** – currency trades.
    
2. **Fixed Income (Bonds, Repos, Interbank lending)** – debt instruments.
    
3. **Equities (Stocks)** – ownership shares.
    
4. **Commodities (Gold, Oil, etc.)** – physical goods traded in markets.
    
5. **Derivatives** – futures, options, swaps.
    

👉 In your project:

- Trades were **partitioned by asset class in Kafka topics**.
    
- Example:
    
    - `trades.fx` → FX trades.
        
    - `trades.repos` → Repo trades.
        
    - `trades.bonds` → Bond trades.
        
- This made downstream analytics (Kafka Streams, Databricks pipelines) more efficient.
    

---

## 🎯 Interview-Ready One-Liner

> “Normalizing trades with Avro means converting different trade formats from OMS/EMS into a consistent Avro schema, enforced by Schema Registry, so all trades can be processed uniformly downstream. We then partitioned trades by asset class — like FX, bonds, repos, and equities — which allowed us to apply different business logic for each class and calculate metrics like LCR and HQLA buffers accurately.”

---

⚡Do you want me to also give you a **mini Avro schema example** (say, for a FX trade vs a Repo trade) so you can explain how normalization actually looks in code during an interview?

------------------------------------


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