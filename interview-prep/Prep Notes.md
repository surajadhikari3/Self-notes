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


