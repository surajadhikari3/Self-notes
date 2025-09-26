
so trades come from different upstreams services like web-apps, mobile-apps, third party oms, brokerages, stop-loss job to OMS(Order Management System) and validations are performed and sent to incoming-ordered kafka topic and the trades goes in 2 ways one is EMS (will come back in a sec) and 

Trade Capture Enrich and Normalization system where the trades are enriched by creating a KStreams and joining on different KTables like customer demographics, security references and counterparty directories and then sent to an enriched-trade kafka topic and then it goes for different downstream services like position keeping engine where transaction is created and entered in journal entry and then the position is upserted in the database and sent to a kafka topic with cdc which are being consumed by Treasury Management System. 

TMS capture and filters the trade that is related with the Liquid Assets like Fixed Income, Equities, Commodities etc. Those trades are landed into the databricks using the kafka topic. We are following the Medallion architecture in the databricks which is layer based and composed of bronze, silver and gold layer. The trade landed into the bronze zone and parked as a delta format . These delta format trade are further processed and imposed the buisness validation in silver layer. Using the  spark structured streaming and dlt pipelines we are calculating the metrics like LCR buffers, HQLA classifications, net cash outflow  with the bitemporality provided by SCD2. These curated massaged data are further aggregated in gold layer and the view is created on top of it. From this view the streaming dashoard on deephaven is created for regular monitoring and analysis of Liquidity coverage Ratio and comparison with net cash outflow..


Also I've worked on the Deephaven and streamlit to build the streaming dashboard for the



coming back to EMS - execution mangement system where the trade is executed based on the SOR- smart order routing engine (which is managed by other group) where best execution strategy is applied and executed on stock exchanges like NASDAQ, NYSE,TSX,S&P500 etc... and send a dispttach venue event and our system listens to those engines, since trades comes in different formats like json,avro , we create a fix format using quickfix/j library and cretes a TCP socket connection with fix brokers and once the trade is executed with either buy,sell,spread, the response is sent back to the EMS. so that's the high level of trade execution we are managing. 













1. **Trade Capture, Enrich, and Normalization (TCEN)**: Trades are enriched using **Kafka Streams**, joining with multiple **KTables** such as customer demographics, security details, and counterparty directories. The enriched trades are then published to an **enriched-trade Kafka topic**, which feeds downstream systems like the Position Keeping Engine. Here, transactions are recorded in the journal, positions are upserted in the database, and changes are published to Kafka topics via **CDC**, consumed by **liquidity desks and settlement ladders**. Additionally, the enriched Kafka topic is connected to **Kafka sink connectors** like **MongoDB and Elasticsearch**, which are used to power **Trade Audit APIs** and **React-based front-end dashboards**, providing real-time visibility into trade data. Any invalid trades are sent to a **dead-letter topic**. 


2. **Execution Management System (EMS)**: In EMS, trades are executed based on the **SOR (Smart Order Routing) engine**, which applies the best execution strategy across exchanges like **NASDAQ, NYSE, TSX, and S&P 500**. Since trades arrive in multiple formats (JSON, Avro), we convert them into **FIX messages** using the **QuickFIX/J library** and establish **TCP socket connections** with FIX brokers. Once executed, the responses (buy, sell, or spread) are sent back to EMS, completing the trade lifecycle. Overall, I manage **end-to-end trade flow**, from capture and enrichment to execution and downstream processing, ensuring reliability, scalability, and integration with multiple data and execution systems, while supporting **real-time analytics through dashboards and APIs**.





“We follow a **contract-first approach** with OpenAPI specs. This way, the request/response models, examples, and error payloads are clearly defined upfront. Our APIs use backward-compatible versioning to avoid breaking consumers. For event-driven services, we use Avro schemas stored in Schema Registry, enforcing compatibility rules (usually BACKWARD) to ensure producers and consumers evolve safely without breaking integration.”


Flow for how do you expose your service

**Consumer → Apigee (API Gateway) (handles authentication OAUTH2/ JWT) → Azure Front Door/App Gateway (WAF, optional for DDOS and threat proction)  → AKS Ingress/Load Balancer → Microservice Pods (ClusterIP)**


> “Our microservices run inside **AKS** as internal `ClusterIP` services. Externally, we expose them through **Apigee**, which serves as our API gateway. Apigee handles API security (OAuth2/JWT, quotas, rate limiting), transformations, and publishes our Swagger contracts to a central developer portal. From Apigee, requests are routed into Azure — optionally via **Front Door/App Gateway with WAF** for DDoS and threat protection — and then into our AKS ingress/load balancer. Inside AKS, the ingress forwards the requests to the correct microservice pods. This setup allows us to decouple external API management from cluster concerns, giving us consistent governance across all APIs while still leveraging AKS for scalability and container orchestration.”




“In Azure, we manage secrets with **Azure Key Vault**. All sensitive credentials (DB passwords, API keys, TLS certs) are stored centrally in Key Vault, never inside code or container images. Our AKS workloads use **managed identities** to authenticate against Key Vault — this avoids hardcoding any credentials. At runtime, we mount or inject secrets into pods using the **Secrets Store CSI Driver**, which fetches secrets directly from Key Vault. Secrets are rotated regularly, and access is enforced via RBAC and least-privilege principles, so only the right service can fetch the right secret.”


> “In production, we never run databases inside pods. We use managed services like **Azure PostgreSQL** or **Cosmos DB**, because they handle backups, HA, and scaling automatically. For development or POCs, if we do run a database inside Kubernetes, we deploy it as a **StatefulSet** with a **PersistentVolume** so data survives pod restarts. The database is exposed via a **ClusterIP service**, so apps can connect to it using internal DNS like `postgres.namespace.svc.cluster.local`. We secure access with TLS and credentials from **Azure Key Vault**, and add retry/backoff logic in the app in case of pod restarts. This way, devs get a self-contained setup, while prod stays on reliable managed services.”


**Handling difficult colleagues?**  

listen first, understand , 
clarify goals ,
does not only rely on the verbal communication instead align on data and acceptance criteria,
propose an experiment behind a feature flag, 
and follow up with measurable outcomes.


**Proposing ideas / what if team rejects it?**

Go through the documentation, understanding and quick implementation  with small pocs, collect the output metrices, numbers, visulization and side by side comparison and documentation in the confluence page . There is lot of factors in the enterprise environment like price, ecosystem so i move forward with the positive attitude and document it for other devs why it was tried and rejected.

Write an ADR with trade-offs, run a small POC, present metrics. If rejected, I document the decision, learnings, and revisit when constraints or data change.



“We measure API performance mainly through **latency and throughput**. For latency, instead of just looking at averages, we monitor **p95 and p99 latencies**, which show the slowest 5% or 1% of requests. For example, most read endpoints respond in a few tens of milliseconds under normal load. For throughput, we scale horizontally by adding more pods in Kubernetes as traffic grows. We also use caching with Redis to keep frequently accessed data in memory, which helps reduce hot-path latency and avoids extra database calls.”


“If API performance degrades, I first check the **golden signals** — latency, traffic, errors, and saturation — to see if it’s a code issue, a sudden traffic spike, or an infrastructure bottleneck. Then I correlate with recent deployments and look at traces and DB metrics to pinpoint the root cause. For fixes, I usually start with a **rollback** if a new release caused it. Otherwise, I use caching to offload hot queries, optimize DB queries and indexes, add bulkheads/circuit breakers to isolate failures, or scale out with more pods. The goal is to stabilize quickly, then dig deeper for long-term improvements.”


> “When running multiple API replicas, we ensure consistency at several layers. On the database side, we rely on **ACID transactions** with `READ COMMITTED` or stricter isolation to prevent dirty reads. For concurrent writes, we use **optimistic locking** so updates don’t overwrite each other. At the application layer, handlers are **idempotent** so retries don’t cause duplication. For DB ↔ Kafka sync, we use the **outbox pattern** so messages and DB changes are committed together. To improve read performance, we use a **cache-aside strategy with TTLs and versioned keys**, and for GET requests we can offload to **read-only replicas** where consistency lag is acceptable. This ensures no dirty reads, no corruption, and consistent updates across replicas.”