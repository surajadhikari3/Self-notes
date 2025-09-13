  
Kafka streams DSL —> Stateless and stateful transformation can be done here**
 **vs Processor Api -> Define the topology and add the source node, other nodes and marked the termination with sink node. —> it lets to create the own DAG(Direct acyclic graph ) for custom use case while designing the stream processing application..** 

  Gobal k table --> is without the partition whereas normal k table is with the partition.......

Consumer producer app  
  
Native kafka client library jar for producing consuming (Just using the java not the spring boot like framework)  
  
Avro —>

 Binary format ( 1 0 )

Faster serilization and desirilizarion since it is binary format 

Save 10 x storage and computing….

Mssg validation against transaction (it compares against the schema registry and validates)

  
100 tps —> About  100 million records process in kafka  

1 million message should process under 5 min 

**Kafka. Streams**

Declarative vs imperative

Topology  
  
Operator in kaka streams  
  
mapValues(values ->  
map(key, value) ->  
filter((key, value) ->  Long.parseValue(value) > 1000)  
  
Kstream -> can subscribe to the multiple topic…  
KTable —> it can only subscribe to the single table  
  
  
serdes -> Serializer/Deserializer…..(Combined form of the serializer and deserializer)  
  
Joins…..

### 🔁 **KStream vs KTable: Key Differences**

| Feature       | `KStream`               | `KTable`                            |
| ------------- | ----------------------- | ----------------------------------- |
| Nature        | Stream of records       | Table with the latest value per key |
| Records       | Append-only (event log) | Keyed updates (changelog stream)    |
| Use Cases     | Event processing        | Aggregation, lookups, joins         |
| Example       | Orders, logs            | Inventory counts, user state        |
| Joins With    | `KStream`, `KTable`     | `KTable`, `KStream`                 |
| Re-processing | All events              | Only latest key-value               |


### 🧠 Real-world Analogy:

| Concept   | Analogy                                                                           |
| --------- | --------------------------------------------------------------------------------- |
| `KStream` | Like a **transaction log** — every line is an action (e.g., "User bought item X") |
| `KTable`  | Like a **database table** — for each user, only the **latest balance** is kept    |


Diagram: Visualizing KStream & KTable Joins

               ┌────────────────────────────┐
               │        KStream             │  ➜ Order Events
               │  Topic: orders             │  e.g., (user1, "order#123")
               └────────────┬───────────────┘
                            │
                            ▼
                 (Stream of Order Events)

     JOIN with 👇

               ┌────────────────────────────┐
               │         KTable             │  ➜ User Profiles (latest only)
               │  Topic: user_profiles      │  e.g., (user1, "Gold Member")
               └────────────┬───────────────┘
                            │
                            ▼
                 (Table of latest user state)

                            │
                            ▼

               ┌────────────────────────────┐
               │   Enriched KStream Output  │
               │  Topic: enriched_orders    │
               │  (user1, order#123 + Gold) │
               └────────────────────────────┘



### **A. KStream → KTable Join**

**Use Case**: Enriching real-time events with user or product metadata.

| Example                                          | Why It’s Useful                                              |
| ------------------------------------------------ | ------------------------------------------------------------ |
| Join orders (KStream) with user profile (KTable) | Add loyalty status, shipping preferences before fulfillment. |
| Join purchase stream with product info table     | Attach price, category, inventory status.                    |

📝 **Code Snippet**:


```
KStream<String, Order> orderStream = builder.stream("orders");
KTable<String, UserProfile> profileTable = builder.table("user_profiles");

KStream<String, EnrichedOrder> enriched = orderStream.join(
    profileTable,
    (order, profile) -> new EnrichedOrder(order, profile)
);

```

---

### ✅ **B. KStream → KStream Join**

**Use Case**: Correlating two event streams within a **time window**.

|Example|Why It’s Useful|
|---|---|
|Join payment events with order events|Detect if payment came within 10 minutes of order.|
|Join app login and location streams|Track user activity and geo-location together.|

📝 **Code Snippet**:

java

CopyEdit

```
KStream<String, Order> orders = builder.stream("orders");
KStream<String, Payment> payments = builder.stream("payments");

KStream<String, Matched> matched = orders.join(
    payments,
    (order, payment) -> new Matched(order, payment),
    JoinWindows.ofTimeDifferenceWithNoGrace(Duration.ofMinutes(5))
);

```

---

### ✅ **C. KTable → KTable Join**

**Use Case**: Building a unified view of system state.

|Example|Why It’s Useful|
|---|---|
|Join user settings with preferences|Build a complete settings profile.|
|Join inventory count with pricing|Dashboard for retail managers.|

📝 **Code Snippet**:


```
KTable<String, Inventory> inventoryTable = builder.table("inventory");
KTable<String, Pricing> pricingTable = builder.table("pricing");

KTable<String, ProductStatus> productStatus = inventoryTable.join(
    pricingTable,
    (inventory, pricing) -> new ProductStatus(inventory, pricing)
);

```

---

## 📌 Summary Table: Join Type Use Cases

|Join Type|Left Side|Right Side|Real-World Example|
|---|---|---|---|
|`KStream-KTable`|Stream|Lookup|Enrich order with user profile|
|`KStream-KStream`|Stream|Stream|Match order and payment within time window|
|`KTable-KTable`|Table|Table|Merge pricing and inventory data|

## 🪟 **Types of Windows in Kafka Streams**

Kafka provides several types of windows depending on how you want to group the events in time:

| Window Type          | Description                                                             | Real-world Example                                  |
| -------------------- | ----------------------------------------------------------------------- | --------------------------------------------------- |
| **Tumbling Window**  | Fixed-size, **non-overlapping** windows                                 | Count page views **per minute**                     |
| **Hopping Window**   | Fixed-size, **overlapping** windows with hop (slide) interval           | Count logins in a **5-min window every 1 min**      |
| **Sliding Window**   | Windows **based on difference between event timestamps** (dynamic size) | Match related events like **click → purchase**      |
| **Session Window**   | **Inactivity-gap-based** dynamic window                                 | User activity session — ends if idle for 30 seconds |
| **Unlimited Window** | Infinite window (default) for non-windowed aggregations                 | Count all-time messages (until app is stopped)      |
## 🔥 **Most Commonly Used Windows**

| Use Case                          | Common Window Type  |
| --------------------------------- | ------------------- |
| Real-time dashboard (rolling avg) | **Hopping Window**  |
| Event correlation                 | **Sliding Window**  |
| Periodic reporting                | **Tumbling Window** |
| User session behavior             | **Session Window**  |
| Lifetime counters                 | **Unlimited**       |
## Visual Overview

### Tumbling vs Hopping vs Sliding

Time Axis ➡

Tumbling:  [-----] [-----] [-----]
Hopping:   [-----]
             [-----]
               [-----]
Sliding:    [---]   [---]   [---]   (based on time difference between two records)


### Acknowledgment + Processing Guarantees

| Guarantee         | Producer Setting                        | Consumer Offset Strategy           |
| ----------------- | --------------------------------------- | ---------------------------------- |
| **At-most-once**  | `acks=0` or `1`                         | Auto-commit (may skip processing)  |
| **At-least-once** | `acks=all`                              | Manual commit **after** processing |
| **Exactly-once**  | `acks=all` + Idempotence + Transactions | Transactions + commit within scope |
## 📌 Summary Table

| Role     | Type          | What It Does                             | Config / API                           |
| -------- | ------------- | ---------------------------------------- | -------------------------------------- |
| Producer | Acks          | Controls write durability to Kafka       | `acks=0/1/all`                         |
| Consumer | Offset Commit | Controls read/processing confirmation    | `enable.auto.commit`, `commitSync()`   |
| Streams  | EOS           | Guarantees once-and-only-once processing | `processing.guarantee=exactly_once_v2` |

## **Consumer Acknowledgements (Offset Commits)**

Kafka doesn’t **auto-track what’s been read** — **consumers commit offsets** to mark progress.

### ✅ Types of Offset Commits:

| Type                    | Description                                                         |
| ----------------------- | ------------------------------------------------------------------- |
| **Automatic commit**    | Offsets are committed at intervals (e.g., every 5 sec)              |
| **Manual commit**       | Developer explicitly commits offset after successful processing     |
| **Synchronous commit**  | `commitSync()` — waits for broker ack, retries on failure           |
| **Asynchronous commit** | `commitAsync()` — doesn’t wait; faster but **may risk offset loss** |


Apache Kafka ensures **fault tolerance** through a combination of **replication, acknowledgments, leader election, and durable storage**. Let’s break this down in a way that’s **interview-ready** and **system-design-friendly**.

---

## ✅ 1. **Replication: The Core of Kafka Fault Tolerance**

- Each Kafka **topic partition** is **replicated** across multiple brokers (nodes).
    
- You define `replication.factor` (e.g., 3) → there will be **1 leader** and **2 followers**.
    
- All writes go to the **leader partition**; followers **replicate data asynchronously**.
    

📌 **If one broker fails**, Kafka can **elect a follower as the new leader**, ensuring continuity.

---

## ✅ 2. **Leader Election for Partitions**

Kafka uses **Zookeeper** (in older versions) or **KRaft** (in newer versions) for **metadata and leader election**:

- If the broker hosting the leader partition crashes:
    
    - Kafka automatically **elects a new leader** from in-sync replicas (ISR).
        
    - Clients redirect to the new leader.
        

This means Kafka is **self-healing** in the face of node failures.

---

## ✅ 3. **Durable Storage with Write-Ahead Log**

Kafka persists all messages to **disk** (write-ahead log):

- Each message is **written to disk** before acknowledgment.
    
- Even if a broker restarts, it can **replay messages from disk**.
    
- Kafka's log is **append-only**, which is fast and resilient.
    

---

## ✅ 4. **In-Sync Replicas (ISR)**

Kafka tracks which replicas are **up-to-date**:

- **ISR** = set of replicas that are fully caught up with the leader.
    
- A message is considered **committed** only if it's replicated to all ISR members (configurable).
    

You can configure **acks**:

- `acks=0` → Fire and forget
    
- `acks=1` → Only leader acknowledges
    
- `acks=all` → All in-sync replicas acknowledge (stronger durability)
    

> ✅ For fault-tolerant systems, use `acks=all`.

---

## ✅ 5. **Producer Retries & Idempotency**

To avoid message loss or duplication:

- **Retries**: Kafka producer retries sending if a broker is temporarily unreachable.
    
- **Idempotent Producer**: Ensures **exactly-once delivery** semantics.
    
    - Enabled by `enable.idempotence=true`
        
    - Avoids duplicates during retries
        

---

## ✅ 6. **Consumer Group Rebalancing**

If a consumer dies, Kafka:

- **Reassigns partitions** to other consumers in the same group.
    
- Ensures **no message loss**, though some duplication can happen if **offset commit** was delayed.
    

---

## ✅ 7. **ZooKeeper/KRaft Resilience**

Kafka (pre-3.0) relies on **ZooKeeper** for cluster state. ZooKeeper itself is replicated and fault-tolerant.

- In **Kafka KRaft mode (post-3.0)**, Zookeeper is removed and Kafka handles metadata quorum via **Raft consensus**.
    

---

## 💡 Summary Table

| Feature              | Fault Tolerance Role                                |
| -------------------- | --------------------------------------------------- |
| Replication          | Ensures multiple copies of data                     |
| Leader Election      | Keeps partition available even if a broker fails    |
| Durable Disk Storage | Messages are recoverable after crashes              |
| ISR & Acks           | Guarantees committed messages are safely replicated |
| Idempotent Producer  | Prevents duplicates during retries                  |
| Consumer Rebalancing | Keeps consumption going even if a consumer dies     |
| ZooKeeper/KRaft      | Manages leader election and cluster state           |

---

Would you like a **diagram showing leader–follower setup and failover** or a **real-world analogy** to explain this in interviews?

------------------------------------------------

Kafka security Layer:

Transport Layer --> mTLS
SASL (Simple Authentication and Security Layer) --> SASL_SSL


--------------------------

Excellent ask 👍 — at senior Java interviews, Kafka questions are rarely just _definition-based_. They’re usually **scenario-driven** to test design, troubleshooting, and production-readiness. Here’s a curated list of **common scenario-based Kafka questions** you should be ready for (with short notes on how to approach).

---

# 🔥 Top Scenario-Based Kafka Questions (Senior Java)

## 1. **High Throughput**

👉 _“Your application must process 2000 TPS of trade events with <100ms latency. How do you design Kafka producers/consumers to handle this?”_

- Talk about **partitions scaling**, **idempotent producers**, **linger.ms / batch.size tuning**, **compression (lz4/zstd)**, and consumer parallelism.
    
- Mention **load testing with kafka-producer-perf-test**.
    

---

## 2. **Exactly Once Delivery**

👉 _“How do you ensure an order is not processed twice in a trading app?”_

- Producer: `enable.idempotence=true`, `acks=all`, `transactional.id`.
    
- Consumer: read within a transaction, commit offsets only if processing succeeds.
    
- Say “we use **EOS (exactly-once semantics)** with Kafka transactions + DB sink.”
    

---

## 3. **Backpressure / Consumer Lag**

👉 _“What if consumers fall behind and lag starts increasing?”_

- Scale **more partitions + consumers**.
    
- Use **max.poll.records**, **fetch.min.bytes** to batch.
    
- Possibly offload heavy logic via **Kafka Streams / Flink**.
    
- Monitor with **lag exporters**.
    

---

## 4. **Reprocessing Data**

👉 _“A bug was found in trade enrichment. How do you reprocess data from Kafka?”_

- Reset offsets with **`kafka-consumer-groups.sh --reset-offsets`**.
    
- Or use **topic compaction / DLQ replay**.
    
- Or store raw trades in **immutable topic (bronze)** → downstream apps can re-read.
    

---

## 5. **Dead Letter Queue (DLQ)**

👉 _“How do you handle poison messages in Kafka?”_

- Retry mechanism (Spring Kafka RetryTemplate, exponential backoff).
    
- After N retries → send to **DLQ topic**.
    
- Monitor DLQ, fix bad data, re-ingest.
    

---

## 6. **Ordering Guarantees**

👉 _“How do you guarantee order of trades per customer?”_

- Use **keyed messages** (`key=customerId`) → all messages go to the same partition.
    
- One consumer per partition.
    
- Warn: order is only guaranteed _within_ a partition, not across partitions.
    

---

## 7. **Security in Trading Apps**

👉 _“How do you secure Kafka in a financial environment?”_

- `SASL_SSL` with **SCRAM or mTLS**.
    
- **ACLs** per service account (OMS writer, Risk reader).
    
- Encryption at rest (disk/KMS) + payload encryption for PII.
    
- **Least privilege** + secret rotation.
    

---

## 8. **Data Loss / Broker Crash**

👉 _“What happens if a broker crashes — do you lose messages?”_

- Messages safe if **replication.factor ≥ 3** + `acks=all`.
    
- If ISR not met → producers block/retry.
    
- Consumer rebalances automatically after leader election.
    

---

## 9. **Schema Evolution**

👉 _“How do you handle schema changes in Kafka messages?”_

- Use **Schema Registry** with Avro/Protobuf/JSON.
    
- Set **compatibility=BACKWARD**.
    
- Producers evolve schema; consumers still read old messages.
    

---

## 10. **Cross Data Center / DR**

👉 _“How would you design Kafka for disaster recovery?”_

- Use **Cluster Linking** or **MirrorMaker 2**.
    
- Topics replicated to DR site.
    
- Keep ACLs, quotas, schemas synced.
    
- Mention **RPO/RTO trade-offs**.
    

---

## 11. **Large Messages**

👉 _“What if you need to send >10MB messages?”_

- Avoid giant messages; instead **chunk/split** or store payload in **S3/DB** and send reference in Kafka.
    
- If must → tune `max.request.size`, `message.max.bytes`.
    

---

## 12. **Multi-tenant Topics**

👉 _“Different teams share the Kafka cluster. How do you isolate them?”_

- **Separate topics/namespaces**, quotas (`producer_byte_rate`, `consumer_byte_rate`).
    
- ACLs per principal.
    
- Possibly **dedicated clusters** for critical apps (trading vs analytics).
    

---

## 13. **Monitoring & Alerting**

👉 _“How do you monitor Kafka in production?”_

- Metrics: **consumer lag**, ISR count, under-replicated partitions, request latency, authentication failures.
    
- Tools: Prometheus + Grafana, Confluent Control Center, LinkedIn Burrow.
    
- Alert when **lag > threshold** or **ISR < RF**.
    

---

## 14. **Idempotency & Retries**

👉 _“What happens if the producer retries a message?”_

- Without idempotence → duplicate trades possible.
    
- With `enable.idempotence=true` → Kafka ensures each message per partition is written once.
    

---

## 15. **Mixing Batch & Real-Time**

👉 _“We have a batch job and a streaming app reading the same topic. How do you design it?”_

- Use **separate consumer groups** (offsets are per group).
    
- Batch job can consume at its pace, streaming app runs continuously.
    

---

✅ **Pro tip for interviews:**  
When asked a scenario, answer in **3 layers**:

1. **Concept** (what Kafka guarantees).
    
2. **Config/code knobs** (producer/consumer/broker settings).
    
3. **Real-world trade-off** (latency vs throughput, cost vs durability).
    

---

Would you like me to **turn these into Q&A flashcards with short “model answers”** (so you can rehearse them quickly like in an interview)?