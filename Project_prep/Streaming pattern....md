
# 📘 **Apache Spark Structured Streaming: Architecture, Fault Tolerance, and Real-Time Flow**

## ✅ What is Structured Streaming?

Structured Streaming is a **high-level streaming API built on top of the Spark SQL engine**. It enables **near real-time** stream processing using **declarative code**, like writing batch jobs.

---

## 🚀 Key Characteristics

|Feature|Description|
|---|---|
|**Near Real-Time**|Micro-batches or continuous mode process new data quickly|
|**Exactly-Once Guarantee**|Through checkpointing and write-ahead logs|
|**Declarative**|Same DataFrame/Dataset APIs as batch|
|**Event-Time Support**|With watermarking and windowing|
|**Fault-Tolerant**|Automatic recovery using checkpoints and lineage|

---

## 🔁 How Structured Streaming Works Internally

### **1. Input Source Abstraction**

Structured Streaming supports sources like:

- Kafka
    
- Delta/Parquet/JSON (via Auto Loader)
    
- Rate (test source)
    

Each input is treated as an **unbounded table**.

### **2. Logical Query Plan**

You define your transformation using **DataFrame API**:

```python
stream = df.groupBy("user").count()
```

→ Internally compiled into a **logical query plan**

### **3. Triggering Execution**

You can run the query using:

- `trigger(ProcessingTime)`
    
- `trigger(once=True)`
    
- `trigger(continuous=True)` (experimental)
    

Each trigger computes a **new batch of data** → transformed → written to sink.

### **4. Sink: Write Mode**

Supports:

- Append
    
- Update
    
- Complete
    

### **5. Fault Tolerance with Checkpointing**

Spark stores:

- **Offsets** (source progress)
    
- **Intermediate state**
    
- **Metadata of executed plans**
    

in a **checkpoint directory**, e.g., `/mnt/checkpoints/my-stream`

---

## 🔄 **End-to-End Fault Tolerance Workflow**

Here’s how Structured Streaming guarantees fault tolerance and exactly-once:

### **Diagram:**

```
[Source] → [Read Offset N] → [Transform] → [Write to Sink]  
              ↓                         ↓  
     [Checkpoint Offset N]    [Checkpoint Sink Write Metadata]
```

### **Mechanisms:**

|Component|Fault-Tolerance Mechanism|
|---|---|
|**Source (Kafka, Auto Loader)**|Stores offsets in checkpoint dir|
|**Stateful Operations**|Stores aggregation state snapshots|
|**Sink (Delta, Kafka)**|Write-ahead log or idempotent writes|
|**Recovery**|On failure → restart job → read checkpointed offset/state → resume|

---

## 🧪 **Structured Streaming Code Example (with Checkpointing)**

### ✅ **Word Count from Kafka (Stateful Aggregation)**

```python
from pyspark.sql.functions import explode, split

# Read stream
lines = (spark.readStream
         .format("kafka")
         .option("kafka.bootstrap.servers", "localhost:9092")
         .option("subscribe", "words")
         .load())

# Transform
words = lines.selectExpr("CAST(value AS STRING)").select(explode(split("value", " ")).alias("word"))
wordCounts = words.groupBy("word").count()

# Write with checkpoint
query = (wordCounts.writeStream
         .format("delta")
         .outputMode("complete")
         .option("checkpointLocation", "/mnt/checkpoints/wordcount")
         .start("/mnt/output"))
```

---

## 🧠 **Conceptual Model of Structured Streaming**

### **Streaming = Continuous Table Append**

Structured Streaming treats data as **a table that grows over time**:

```text
Input Table
+-----+
|Data |
+-----+
|Row 1|
|Row 2|
|Row 3|   ← New rows keep coming
```

The query processes this as if it were operating on a batch DataFrame.

---

## ⏱️ Trigger Options

|Trigger|Behavior|
|---|---|
|`default`|As fast as possible (micro-batch)|
|`trigger(once=True)`|One-time batch (batch-like + streaming logic)|
|`trigger(processingTime="5 minutes")`|Fixed interval|
|`trigger(continuous=True)`|True streaming (low-latency, experimental)|

---

## 💾 **Output Modes**

|Mode|When to Use|Supported Sinks|
|---|---|---|
|**Append**|Only new rows|All sinks|
|**Update**|Aggregations (state updated)|Console, Kafka, Delta|
|**Complete**|Rewrites full output|Console, Delta|

---

## 🧷 **State Management in Structured Streaming**

### Scenarios:

- Running totals (e.g., word counts)
    
- Windowed aggregations
    

### State Store:

- Backed by local RocksDB + checkpoint
    
- Automatically managed by Spark
    

---

## 💡 Real-World Use Cases

|Use Case|Description|
|---|---|
|**IoT Pipeline**|Ingest real-time sensor data from Kafka, aggregate in windows|
|**Clickstream Analytics**|Session-based metrics using session windows|
|**Fraud Detection**|Real-time join and rule-based alerts|
|**ETL on Files**|Use Auto Loader for structured streaming from cloud storage|

---

## 🧾 Diagram: Streaming with End-to-End Fault Tolerance

```plaintext
       +------------+           +-----------------+         +-------------+
       | Kafka/Files|  ----->   | Structured Stream|  --->  |  Delta Sink |
       +------------+           |  Query Execution |         +-------------+
                                      |
                           +----------v----------+
                           |  Checkpoint Dir     |
                           | (Offsets, State, etc)|
                           +----------------------+
```

---

## 🛡️ How Structured Streaming Ensures Fault Tolerance

|Stage|Recovery Guarantee|
|---|---|
|**Source Offset**|Resumes from last committed offset (no reprocessing)|
|**Stateful Ops**|Recovers aggregation state from checkpoints|
|**Sink Writes**|Idempotent (Delta), Atomic (Kafka), or WAL (HDFS)|
|**Job Restart**|Auto-resumes from failure|

---

## 🛠️ Best Practices

- Use Delta as sink for exactly-once semantics
    
- Set proper watermark to prevent state buildup
    
- Always use checkpointing path
    
- Tune `maxOffsetsPerTrigger` and `trigger` interval for performance
    
- For fault tolerance → **never delete checkpoint dir**
    

---

## ⚙️ Config Parameters to Know

|Parameter|Purpose|
|---|---|
|`spark.sql.streaming.checkpointLocation`|Track offset + state|
|`spark.sql.shuffle.partitions`|Tune parallelism|
|`spark.sql.streaming.stateStore.providerClass`|RocksDB|
|`spark.streaming.backpressure.enabled`|Enable backpressure|
|`spark.streaming.kafka.maxRatePerPartition`|Rate limiting|

---

## 📦 Want More?

I can now generate:

- 🖼️ Visual diagrams as image files (flowcharts, dataflow)
    
- 📝 Markdown/Confluence copy-paste content
    
- 🔧 Spark Job JSON templates for production
    

Would you like a **visual infographic of this full architecture** like before?




# 🔄 **1. Micro-Batch Mode (Default)**

# ⚡ **2. Continuous Processing Mode (Low-Latency, Experimental)**

Let me break both down and then focus deeply on **Continuous Processing** as you requested.

---

## ✅ **Micro-Batch Mode (Default)**

### 🔹 Description:

- Spark collects data in **micro-batches** and processes them periodically.
    
- Default mode used in Structured Streaming jobs.
    

### 🔹 Example:

```python
df.writeStream.trigger(processingTime='1 minute').start()
```

### 🔹 Characteristics:

|Feature|Description|
|---|---|
|Latency|Low (~100s of ms to few sec)|
|Throughput|High|
|Fault Tolerance|Fully supported (checkpointing, recovery)|
|Supported Sinks|All (Delta, Kafka, etc.)|

---

## ⚡ **Continuous Processing Mode** _(Experimental)_

### 📌 Description:

- Introduced in Spark 2.3+
    
- Processes data **row-by-row** as it arrives.
    
- Designed for **ultra low-latency use cases**.
    

> ⚠️ Still experimental and supports a limited set of operations and sinks.

---

### 📐 Diagram:

```
[Event Stream] → [Continuous Query] → [Sink]
       ↓                  ↓              ↓
     (row)           (process)     (flush/write)
```

---

## 🔍 Key Differences: Micro-Batch vs Continuous

|Feature|Micro-Batch|Continuous Processing (Experimental)|
|---|---|---|
|**Processing Granularity**|Micro-batches (rows grouped)|Row-by-row|
|**Latency**|Low (100ms - few sec)|Very low (~1ms)|
|**Supported Triggers**|`processingTime`, `once`|`trigger(continuous="5 seconds")`|
|**Output Modes**|All (append, update, complete)|**Only append**|
|**Stateful Operations**|Supported|**Limited or not supported**|
|**Sinks Supported**|All sinks|Only file, Kafka (append only)|
|**Fault Tolerance**|✅ Full|✅ Exactly-once, WAL-backed|

---

## ✅ When to Use Continuous Processing?

**Use it when:**

- Ultra-low latency is critical (sub-second)
    
- You're doing simple, stateless transformations
    
- You're writing to supported sinks (e.g., Kafka)
    

---

## 🧪 **Code Example (Kafka Continuous Processing)**

```python
df = (spark.readStream
      .format("kafka")
      .option("kafka.bootstrap.servers", "localhost:9092")
      .option("subscribe", "input_topic")
      .load())

df.selectExpr("CAST(value AS STRING)") \
  .writeStream \
  .format("console") \
  .trigger(continuous="5 seconds") \
  .option("checkpointLocation", "/tmp/checkpoints") \
  .start()
```

---

## 🧠 Internals of Continuous Processing

|Component|Mechanism|
|---|---|
|**Execution Engine**|Completely rewritten for row-by-row processing|
|**Fault Tolerance**|Achieved via **write-ahead logs** (WAL) + checkpointing|
|**Backpressure**|Automatically controls ingestion to match sink speed|
|**Watermark / Windows**|Not supported|

---

## 🚫 Limitations of Continuous Processing (as of Spark 3.x)

- No support for:
    
    - Aggregations (groupBy)
        
    - Windows / Watermarking
        
    - Update / Complete output modes
        
    - State store usage
        
- Only **append mode** allowed
    
- Limited sink support (Kafka, File, Console)
    

---

## 📊 Summary Table

|Feature|Micro-Batch|Continuous|
|---|---|---|
|Fault Tolerance|✅|✅|
|Aggregations|✅|❌|
|Windows/Watermarks|✅|❌|
|Latency|~100ms–secs|~1ms|
|Use Case|ETL, dashboards, aggregations|Stateless, ultra low-latency stream|

---

## 🔧 Best Practices

- Use `continuous` **only when latency <100ms is essential**
    
- Use **append mode only**
    
- Use only **Kafka or File sinks**
    
- Avoid any **groupBy, stateful, or complex transformations**
    

---

## 🧾 Final Note

Apache Spark’s **Continuous Mode** is **powerful but limited**, and should be chosen only if:

- Your logic is simple
    
- Latency must be ultra low
    
- You accept the limitations
    

For most production pipelines, **Micro-Batch + Auto Loader + Watermarking** gives you:  
✅ Latency  
✅ Fault tolerance  
✅ Flexibility

---



---FOREACH batch

---

# 🧩 Apache Spark Structured Streaming: `forEachBatch` Pattern

## ✅ **What is `forEachBatch`?**

`forEachBatch()` is a method that allows you to apply **custom batch logic** to each micro-batch in a Structured Streaming query.

It gives you the **full control of a batch DataFrame** for each micro-batch of the stream.

---

## 🎯 **When to Use `forEachBatch`**

|Use Case|Why `forEachBatch` is ideal|
|---|---|
|Complex writes (MERGE/UPSERT)|`writeStream.format(...).start()` doesn’t support `MERGE INTO`|
|Writing to multiple sinks|You can define logic to split and write|
|Interacting with JDBC or REST APIs|You can connect to external systems in batch|
|Conditional processing|You can apply logic based on data inside the micro-batch|
|Custom transformations|Perform deduplication, auditing, alerting, etc.|

---

## 🧠 **Conceptual Flow of `forEachBatch`**

### 🖼️ Diagram:

```
[Streaming Source]
       ↓
[Micro-Batch DataFrame] → forEachBatch((df, batchId) => {
                             Custom Logic Here
                          })
       ↓
[Sink A], [Sink B], [External System]
```

---

## 💡 **Real-World Use Case**

### 🚀 Use Case: Streaming Kafka → Delta Lake with Upsert (MERGE)

**Problem:** Default `writeStream` cannot perform upserts.  
**Solution:** Use `forEachBatch` with Delta `MERGE INTO`.

---

## 🧪 **Code Example: MERGE INTO using `forEachBatch`**

```python
from delta.tables import DeltaTable

def upsert_to_delta(microBatchDF, batchId):
    deltaTable = DeltaTable.forPath(spark, "/mnt/output/target")

    (deltaTable.alias("t")
     .merge(microBatchDF.alias("s"), "t.id = s.id")
     .whenMatchedUpdateAll()
     .whenNotMatchedInsertAll()
     .execute())

# Read from Kafka
streamDF = (spark.readStream
            .format("kafka")
            .option("kafka.bootstrap.servers", "localhost:9092")
            .option("subscribe", "events")
            .load())

# Extract value & parse JSON
jsonDF = streamDF.selectExpr("CAST(value AS STRING) as json")
parsedDF = spark.read.json(jsonDF.rdd.map(lambda x: x.json))

# Apply upsert logic on each batch
query = (parsedDF.writeStream
         .foreachBatch(upsert_to_delta)
         .option("checkpointLocation", "/mnt/chk/upsert")
         .start())
```

---

## 🛠️ **Parameters Passed**

|Parameter|Description|
|---|---|
|`microBatchDF`|A DataFrame representing a micro-batch of the stream|
|`batchId`|A unique long identifier for the batch|

---

## 🧾 **Alternative Example: Streaming + JDBC Sink**

```python
def write_to_mysql(df, batchId):
    (df.write
       .format("jdbc")
       .option("url", "jdbc:mysql://localhost/db")
       .option("dbtable", "events_table")
       .option("user", "root")
       .option("password", "secret")
       .mode("append")
       .save())

streamDF.writeStream.foreachBatch(write_to_mysql).start()
```

---

## ⚙️ Benefits of `forEachBatch`

✅ Allows custom logic (e.g. joins, merges, API calls)  
✅ Full DataFrame API within each micro-batch  
✅ Can perform atomic operations like transactions  
✅ Enables batch logic on streaming data

---

## 🧊 Limitations

|Limitation|Details|
|---|---|
|Must handle idempotency|Since Spark may retry a batch|
|No automatic schema evolution|Like Delta Auto Loader|
|Must manage **merge** targets manually|Not auto-handled|

---

## 📋 Summary Table

|Pattern|`forEachBatch`|
|---|---|
|Type|Micro-batch pattern|
|Output Control|Full control over each batch|
|Suitable for|Upsert, conditional logic, multi-sink|
|Not Ideal for|Ultra low-latency (use continuous mode)|

---

## 🖼️ Combined Data Flow Diagram

```
+---------------------+
|   Kafka / Files     |
+----------+----------+
           |
           v
+---------------------+      forEachBatch((df, batchId) => {
|  Structured Stream  |  →     Custom Logic (merge, JDBC, REST)
+----------+----------+        Write to multiple sinks
           |
           v
+---------------------+
|      Delta Lake     |
|     JDBC / Kafka    |
+---------------------+
```

---

FOREACH BATCH

---

# 🔁 **Apache Spark Structured Streaming – `forEachBatch` Pattern**

---

## ✅ **Description**

`forEachBatch` is a method in **Structured Streaming** that allows **custom logic** to be applied to **each micro-batch** of streaming data as a standard **Spark batch DataFrame**.

It bridges the gap between **streaming ingestion** and **flexible batch-style processing**, enabling full access to Spark transformations and actions.

> ⚡ This is especially useful when you need capabilities like `MERGE INTO`, JDBC writes, multi-sink logic, or conditional branching — all of which are difficult or impossible using `writeStream.format(...).start()`.

---

## 🖼️ **Diagram**

```
+-------------------+
|  Streaming Source |
+--------+----------+
         |
         v
+----------------------------+
|  Micro-Batch DataFrame     |
|  forEachBatch((df, id) => {|
|    Custom Logic Here       |
|  })                        |
+--------+---------+---------+
         |         |
     +---v---+  +--v----+
     | Sink A|  | Sink B|
     +-------+  +-------+
```

---

## 🎯 **Use Cases**

|Scenario|Why use `forEachBatch`|
|---|---|
|**Delta Lake Upserts (MERGE)**|`writeStream` can’t perform `MERGE INTO`|
|**Writing to Multiple Sinks**|Write to S3, Delta, Kafka, etc. in the same batch|
|**Calling External APIs**|Call REST APIs, databases, or send alerts|
|**Deduplication Logic**|Run `dropDuplicates()` or use window + rank|
|**Dynamic Routing**|Branch data conditionally by content|
|**Enrichment Joins**|Join streaming data with static tables before writing|

---

## 💡 **Simple Code Example**

```python
from pyspark.sql import SparkSession
from delta.tables import DeltaTable

spark = SparkSession.builder.getOrCreate()

def custom_batch_logic(df, batchId):
    # Example: Upsert into Delta table
    target = DeltaTable.forPath(spark, "/mnt/delta/target")

    (target.alias("t")
     .merge(df.alias("s"), "t.id = s.id")
     .whenMatchedUpdateAll()
     .whenNotMatchedInsertAll()
     .execute())

# Read Kafka stream
kafka_df = (spark.readStream
            .format("kafka")
            .option("kafka.bootstrap.servers", "localhost:9092")
            .option("subscribe", "input_topic")
            .load())

# Extract value column as string
json_df = kafka_df.selectExpr("CAST(value AS STRING) as raw_json")

# Parse JSON into schema (e.g., id, name, timestamp)
parsed_df = spark.read.json(json_df.rdd.map(lambda x: x.raw_json))

# Use forEachBatch to apply custom logic
(parsed_df.writeStream
 .foreachBatch(custom_batch_logic)
 .option("checkpointLocation", "/mnt/checkpoints/upsert_logic")
 .start())
```

---

## 🧠 Notes & Best Practices

|Tip|Description|
|---|---|
|Handle idempotency|`forEachBatch` may retry the same batch after failure|
|Use `batchId` for logging or checkpoint validation||
|Avoid expensive operations like `.collect()` inside the batch function||
|Always define a reliable schema if reading from JSON or raw source||

---

## 🧾 Summary

|Feature|Description|
|---|---|
|**Pattern**|`forEachBatch`|
|**Granularity**|Per micro-batch|
|**Control**|Full access to batch transformations|
|**Fault Tolerance**|Inherited from Spark's structured streaming checkpointing|
|**Supported Sinks**|All (Delta, JDBC, Kafka, REST, etc.)|
|**Ideal For**|MERGE, multi-sink writes, external integrations|

---

------------------------------------------------------------

---

# 🔁 **Databricks Spark Structured Streaming – `forEachBatch` Pattern (File Source)**

---

## ✅ **Description**

`forEachBatch` in Structured Streaming provides a **powerful control point** to apply **custom batch logic** to each **micro-batch** of data as it is ingested from a streaming source — such as **files landing in a directory** (e.g., `/mnt/raw/`).

Unlike regular `.writeStream.format(...).start()`, the `forEachBatch` method enables:

- Complex batch operations (e.g., **MERGE INTO**, **deduplication**, **validation**)
    
- **Multi-sink writes**
    
- **Calling external systems** (e.g., JDBC, APIs)
    

This pattern is ideal when ingesting files into **bronze → silver → gold** tables or implementing **upserts** with Delta Lake.

---

## 🖼️ **Diagram – forEachBatch with File Source**

```plaintext
+------------------------+
| File Source (CSV/JSON)|
|   /mnt/raw/           |
+----------+------------+
           |
     +-----v-----+
     | Auto Loader|
     | or readStream |
     +-----+-----+
           |
           v
+------------------------+
| forEachBatch((df, id) => { |
|   custom logic per batch   |
| })                         |
+-----+----------------------+
      |
      v
+------------------+
| Delta Lake Sink  |
| JDBC Sink / API  |
+------------------+
```

---

## 🎯 **Use Cases**

|Scenario|Why `forEachBatch` Is Ideal|
|---|---|
|**File-based CDC Updates**|Merge file-based change logs into Delta tables|
|**Multi-stage ETL**|Clean/transform incoming files into silver tables|
|**Custom Filtering or Deduplication**|Apply logic like dropDuplicates()|
|**Audit Logging & Alerting**|Trigger alerts or logs for specific records|
|**Upserts to Golden Tables**|`MERGE INTO` for target Delta table using batch logic|

---

## 💡 **Simple Example: Ingest Files → Merge into Delta Table**

```python
from delta.tables import DeltaTable
from pyspark.sql.functions import col

# 1. Define upsert logic
def merge_batch(microBatchDF, batchId):
    delta_target = DeltaTable.forPath(spark, "/mnt/delta/target_table")

    (delta_target.alias("t")
     .merge(
         microBatchDF.alias("s"),
         "t.id = s.id"
     )
     .whenMatchedUpdateAll()
     .whenNotMatchedInsertAll()
     .execute())

# 2. Read incoming files (streaming source)
raw_stream = (spark.readStream
              .format("cloudFiles")  # use 'csv' or 'json' for simple readStream
              .option("cloudFiles.format", "csv")
              .option("header", "true")
              .schema("id STRING, name STRING, updated_at TIMESTAMP")
              .load("/mnt/raw/cdc-files/"))

# 3. Apply forEachBatch with merge logic
query = (raw_stream.writeStream
         .foreachBatch(merge_batch)
         .option("checkpointLocation", "/mnt/checkpoints/cdc_merge/")
         .start())
```

---

## ⚙️ Key Considerations

|Point|Details|
|---|---|
|`microBatchDF`|A full DataFrame of newly arrived file rows|
|`batchId`|A long value for the micro-batch ID (can be used for auditing)|
|Checkpointing|Required to avoid duplicate processing after failures|
|Schema|Must be explicitly defined for file formats like CSV or JSON|
|Idempotency|Recommended to avoid duplication if batch retries|

---

## 📌 Summary

|Feature|Value|
|---|---|
|**Source**|File-based (CSV, JSON, Parquet)|
|**Pattern**|`forEachBatch((df, batchId) => {})`|
|**Control**|Full DataFrame access per micro-batch|
|**Write Support**|Delta Lake, JDBC, REST, multi-sink|
|**Best Use Case**|Merge, dedup, enrichment, complex ETL|

---

Would you like a **PNG diagram** for this workflow or a **Confluence/Markdown export** of the content?

I can also generate advanced examples (multi-sink, schema evolution, audit table writes) if needed.