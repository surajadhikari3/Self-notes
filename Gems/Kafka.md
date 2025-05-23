  
Kafka streams DSL —> Stateless and stateful transformation can be done here**
 **vs Processor Api -> Define the topology and add the source node, other nodes and marked the termination with sink node. —> it lets to create the own DAG(Direct acyclic graph ) for custom use case while designing the stream processing application..** 

  

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
  
Kstream -> can subscripe to the multiple topic…  
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

|Example|Why It’s Useful|
|---|---|
|Join orders (KStream) with user profile (KTable)|Add loyalty status, shipping preferences before fulfillment.|
|Join purchase stream with product info table|Attach price, category, inventory status.|

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