
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

Dashboard stuff................

# 📊 Databricks Dashboard Documentation

## 1. Overview

**Databricks Dashboards** are interactive visual interfaces that allow users to visualize data using SQL queries or Delta tables directly within the Databricks workspace. Dashboards help teams monitor metrics, detect anomalies, and derive insights without needing to export data to external BI tools.

Dashboards are tightly integrated with the **Databricks Lakehouse platform**, enabling seamless access to data through **Unity Catalog**, and supporting advanced governance, sharing, and automation.

---

## 2. How Dashboards Integrate with Unity Catalog

### 📌 Description:

Databricks dashboards rely on SQL queries, which in turn access data stored in tables and views governed by **Unity Catalog**. Unity Catalog provides **fine-grained access control**, **auditing**, and **data lineage**, ensuring that dashboard users only see what they are allowed to access.

### 🧠 How it works:

- The dashboard queries reference **catalog.schema.table** format.
    
- Access control is enforced at the catalog level, ensuring secure and governed access to data.
    
- When a user accesses a dashboard, Unity Catalog evaluates their permissions before returning results.
    

### ✅ Use Case:

A business analyst creates a dashboard summarizing sales data across regions. The data is stored in `main.sales_db.sales_summary`. Unity Catalog ensures:

- The analyst only sees tables they have permission to access.
    
- Unauthorized users cannot view or modify the dashboard unless explicitly granted access.
    

### 💡 Example:

```sql
SELECT region, SUM(revenue)
FROM main.sales_db.sales_summary
GROUP BY region;
```

This SQL block powers a pie chart in a dashboard panel. The access to `main.sales_db.sales_summary` is validated by Unity Catalog.

---

## 3. Impact of Changes in Underlying Data Sources

### 📌 Description:

If the underlying **data source/table/view/schema** changes, dashboards may be impacted based on the **type of change**.

### ⚠️ Types of Changes and Their Impacts:

|Change Type|Impact on Dashboard|
|---|---|
|Table schema change (e.g. drop column)|SQL queries may break or return incorrect results|
|Table/view renamed or deleted|Dashboard panel will fail to load|
|Permissions changed|Users may see "Access Denied" or missing data|
|Data format/type changes|Visualizations might display inconsistently|
|Data content change|Charts may show updated values (real-time or refresh-based)|

### 🧠 Use Case:

If a data engineer renames a column used in a dashboard query:

```sql
SELECT old_column_name FROM table_x
```

The dashboard panel using this query will throw an error until updated.

### ✅ Best Practices:

- Use **views** instead of raw tables to provide abstraction.
    
- Communicate schema changes via release notes.
    
- Test dashboards post-migration or after major ETL updates.
    

---

## 4. Dashboard Sharing, Embedding, and Access Permissioning

### 📌 Description:

Databricks dashboards can be **shared** within the workspace, **embedded** externally, and protected using **granular permission settings**.

---

### 🔒 Access Permissioning:

- Dashboards follow **workspace object permissions**:
    
    - **Can View** – view-only access
        
    - **Can Run** – view + refresh
        
    - **Can Edit** – modify queries, visuals
        
    - **Can Manage** – full control including delete & permissions
        
- Unity Catalog also restricts access to the **underlying data** in the dashboard via table/view permissions.
    

**Note:** Having access to the dashboard **does not imply access to the data** inside it unless granted via Unity Catalog.

---

### 🤝 Sharing Options:

- **Within Workspace**: Share with users or groups in Databricks.
    
- **Public Sharing** _(if allowed)_: Embed via iframe for non-authenticated access (workspace admin-controlled).
    
- **Email Subscription**: Schedule dashboards to email results as PDF or static image.
    

---

### 🌐 Embedding:

- Dashboards can be embedded in external apps via iframe (if public sharing is enabled).
    
- Embedding requires:
    
    - Enabling **public access to the dashboard**
        
    - Admin toggling **"Allow Public Sharing"** setting
        

---

### ✅ Use Case:

A project manager wants a real-time executive overview shared with leadership. They:

1. Create a dashboard.
    
2. Set SQL queries to auto-refresh.
    
3. Share with stakeholders (view-only).
    
4. Schedule daily emails at 8 AM.
    
5. Optionally embed into internal wiki via iframe.
    

---

## 5. Example: Dashboard Setup & Usage

1. **Create Dashboard**:
    
    - Click “+ Create” → Dashboard
        
    - Add a query:
        
        ```sql
        SELECT product, SUM(sales) as total FROM main.retail_db.sales GROUP BY product
        ```
        
    - Choose chart: Bar Chart
        
2. **Set Permissions**:
    
    - Add TeamX with "Can View"
        
    - Add Analyst with "Can Edit"
        
3. **Embed / Share**:
    
    - Click “Share”
        
    - Toggle "Allow Public Access"
        
    - Copy iframe code or set up email schedule
        

---

## 6. Summary

|Feature|Description|
|---|---|
|Integration with Unity Catalog|Ensures secure, governed access to data in dashboards|
|Data source changes|May break dashboards if schema/tables/views are modified|
|Sharing|Share with specific users/groups, or publicly with proper admin config|
|Embedding|Enable public embedding via iframe for external visualization|
|Permissioning|Dashboard + data-level access managed separately via object ACLs + catalog|

-----------------------------------------------------------------


Dashbaord

---

## ✅ Best Practices for Schema Evolution in Retail Revenue & Supply Chain Dashboards in Databricks

These built-in dashboards use **Delta Lake**, **Unity Catalog**, and **SQL/Dashboards** to visualize real-time metrics like revenue, order count, stock levels, and fulfillment KPIs.

When you're **experimenting, updating ingestion pipelines, or evolving schemas**, follow this strategy:

---

### ⚙️ 1. **Allow Schema Evolution ONLY at the Bronze Layer**

If you're ingesting from Autoloader or streaming CSV/Parquet files into Delta, enable schema evolution there:

```python
df.write.format("delta") \
  .option("mergeSchema", "true") \
  .mode("append") \
  .save("/mnt/retail/bronze/retail_sales_raw")
```

> 🟨 **Use `mergeSchema` ONLY here**, because dashboards don’t point to this raw data.

---

### 🧽 2. **Control Schema in Silver Layer (Refined Tables)**

When transforming from Bronze to Silver (`retail_sales_clean`, `orders_refined`, etc.), use **explicit schema projection**:

```python
df.select("order_id", "customer_id", "order_date", "store_id", "product_id", "quantity", "revenue")
  .write.format("delta")
  .mode("overwrite")
  .saveAsTable("retail.revenue.orders_cleaned")
```

> ✅ Prevents new fields from “leaking” into the refined layer without review.

---

### 👓 3. **Expose Data to Dashboard via Stable Views Only**

Create or replace SQL views with fixed schema, like:

```sql
CREATE OR REPLACE VIEW retail.revenue.dashboard_sales_view AS
SELECT
  order_id,
  store_id,
  product_id,
  revenue,
  order_date
FROM retail.revenue.orders_cleaned;
```

- 🛡️ This acts as a **contract**: dashboards only see expected columns.
    
- New columns added during evolution will not affect this view until explicitly updated.
    

---

### 🧠 4. **Use Dashboard-View Versioning for Stability and Dev Testing**

|View Name|Purpose|
|---|---|
|`dashboard_sales_view_v1`|Used by production dashboard|
|`dashboard_sales_view_latest`|Shows evolving schema for analysts/testers|

- Production dashboards **always point to `v1`** (no breakage).
    
- You can inspect new fields in `latest` without breaking visuals.
    

---

### 🔍 5. **Run Schema Drift Checks Periodically (Optional Notebook)**

Write a notebook to **compare current base table schema vs. dashboard view schema**.

```python
df_table = spark.table("retail.revenue.orders_cleaned").schema
df_view  = spark.table("retail.revenue.dashboard_sales_view").schema

if df_table != df_view:
    print("⚠️ Schema has changed — review required!")
```

- 📢 Alert the dev team if new fields appear.
    
- Prevents silent breaking of dashboards.
    

---

### 🔐 6. **Restrict Schema Changes on Silver/Gold Tables with Unity Catalog**

Use Unity Catalog to **block accidental drops or changes** on tables/views used in dashboards:

```sql
-- Example
GRANT SELECT ON TABLE retail.revenue.orders_cleaned TO analysts;
DENY ALTER ON TABLE retail.revenue.orders_cleaned TO analysts;
```

> ❌ Prevent ALTER/REPLACE by non-admins  
> ✅ Ensures dashboard stability

---

### 🖼️ 7. **Don't Use `SELECT *` in Views/Notebooks Feeding Dashboards**

Avoid dynamic schema in views:

```sql
-- ❌ Risky
CREATE OR REPLACE VIEW dashboard_view AS SELECT * FROM orders_cleaned;

-- ✅ Safe
CREATE OR REPLACE VIEW dashboard_view AS
SELECT order_id, store_id, revenue FROM orders_cleaned;
```

---

## 📘 Summary Table (for Retail Revenue Dashboard)

|Layer|Strategy|
|---|---|
|Bronze|Allow schema evolution (`mergeSchema = true`)|
|Silver/Gold|Use fixed schema (explicit `select(...)`)|
|Views|Create stable views (`dashboard_sales_view`) with only required fields|
|Dashboards|Point to stable views only|
|Versioning|Maintain `v1`, `latest`, or staging versions of views|
|Monitoring|Use notebook to detect schema drift|
|Access Control|Use Unity Catalog to deny unauthorized schema changes|

---

## 🛠️ Example: In the Context of Retail Revenue Dashboard

Let’s say the ingestion team adds a new column `delivery_partner_id` to `orders_cleaned`.

### 🚫 What happens if dashboard points to table?

- Dashboard **breaks** if filters or visualizations depend on old schema.
    

### ✅ If dashboard uses view:

- View remains stable.
    
- You can **test the new column in `dashboard_sales_view_latest`**.
    
- Once confirmed, update the stable view `dashboard_sales_view_v1`.
    

---

Let me know if you want:

- A **template view manager notebook**
    
- A **schema audit tracker notebook**
    
- Sample **CI checks for schema diff**
    

I’ll share code snippets tailored to your demo environment.



---code

---

Certainly! Here's the **complete Databricks notebook code** as a single copy-pasteable snippet, showcasing **real-time dashboard best practices** using `samples.nyctaxi.tripdata`.

---


Here's the full **updated code snippet** for your Databricks notebook, designed to demonstrate end-to-end best practices for a real-time dashboard using the `samples.nyctaxi.trips` dataset — with only Bronze layer schema evolution.

---

### 🚖 NYC Taxi Trip Data – Real-Time Dashboard with Schema Best Practices

#### 🔁 Step 1: Ingest to Bronze Layer (Schema Evolution ON)

```python
# Simulated Bronze Ingestion from sample Delta table
bronze_path = "/mnt/bronze/nyctaxi_trips"

raw_df = spark.read.table("samples.nyctaxi.trips")

raw_df.write.format("delta") \
    .option("mergeSchema", "true") \
    .mode("append") \
    .save(bronze_path)
```

---

#### 🧽 Step 2: Create Cleaned Silver Table (Fixed Schema)

```python
from pyspark.sql.functions import col

silver_table = "main.nyctaxi.trips_cleaned"

bronze_df = spark.read.format("delta").load(bronze_path)

clean_df = bronze_df.select(
    col("tpep_pickup_datetime"),
    col("tpep_dropoff_datetime"),
    col("trip_distance"),
    col("fare_amount"),
    col("pickup_zip"),
    col("dropoff_zip")
)

clean_df.write.format("delta") \
    .mode("overwrite") \
    .saveAsTable(silver_table)
```

---

#### 👓 Step 3: Create Stable View for Dashboard

```python
stable_view = "main.nyctaxi.trips_dashboard_view"

spark.sql(f"""
CREATE OR REPLACE VIEW {stable_view} AS
SELECT
  tpep_pickup_datetime,
  tpep_dropoff_datetime,
  trip_distance,
  fare_amount,
  pickup_zip,
  dropoff_zip
FROM {silver_table}
""")
```

---

#### 🧪 Step 4: (Optional) Create Dev/Dynamic View

```python
latest_view = "main.nyctaxi.trips_dashboard_view_latest"

spark.sql(f"""
CREATE OR REPLACE VIEW {latest_view} AS
SELECT * FROM {silver_table}
""")
```

---

#### 🛡️ Step 5: Schema Drift Detection (Optional)

```python
silver_schema = set((f.name, str(f.dataType)) for f in spark.table(silver_table).schema)
view_schema = set((f.name, str(f.dataType)) for f in spark.table(stable_view).schema)

added = silver_schema - view_schema
removed = view_schema - silver_schema

if added or removed:
    print("⚠️ Schema Drift Detected")
    if added: print("➕ New columns in table:", added)
    if removed: print("➖ Missing columns in table:", removed)
else:
    print("✅ Schema matches view")
```

---

#### 📊 Step 6: Build Dashboard (Manually in Databricks SQL)

Use the view `main.nyctaxi.trips_dashboard_view` and create visuals such as:

- Total fare by `pickup_zip`
    
- Avg. trip distance by `dropoff_zip`
    
- Line chart of trips by `tpep_pickup_datetime`
    
- Filters on ZIP codes and pickup/dropoff time
    

---


Best practises Documentation ................


---

# 🚖 Real-Time Dashboard Implementation with Schema Best Practices in Databricks

This documentation outlines how a production-grade dashboard was built using **Databricks best practices**, based on the `samples.nyctaxi.trips` dataset. The pipeline follows a proper **Bronze → Silver → View → Dashboard** architecture with **schema control** and **observability** in place.

---

## ✅ Architecture Overview

> **Data Flow:**  
> Shared Dataset → Bronze Layer (Delta with schema evolution) → Silver Layer (fixed schema) → Views → SQL Queries → Visual Dashboard

![Architecture Diagram Placeholder](https://chatgpt.com/g/g-p-680bf5618d8481918b06e8101d344bea-suraj/c/insert-architecture-diagram)

---

## 🟫 Step 1: Ingest to Bronze Table with Schema Evolution

- Ingested raw trip data from `samples.nyctaxi.trips`
    
- Wrote data to `/mnt/bronze/nyctaxi_trips` using Delta format
    
- Enabled schema evolution (`mergeSchema=true`) to support ingestion flexibility
    

📌 **Purpose:** Allow evolving raw schemas (e.g., new fields) without blocking the pipeline

```python
raw_df = spark.read.table("samples.nyctaxi.trips")

raw_df.write.format("delta") \
    .option("mergeSchema", "true") \
    .mode("append") \
    .save("/mnt/bronze/nyctaxi_trips")
```

📸 _Screenshot: Code cell or notebook output_  
![Bronze Layer Screenshot](https://chatgpt.com/g/g-p-680bf5618d8481918b06e8101d344bea-suraj/c/insert-image-here)

---

## 🪙 Step 2: Clean and Load Silver Table (Fixed Schema)

- Read data from Bronze path
    
- Selected only known, required columns (e.g., pickup/dropoff time, fare, ZIPs)
    
- Wrote output to `main.nyctaxi.trips_cleaned` table using Delta format
    

📌 **Purpose:** Enforce schema control and filter out unnecessary or risky fields

```python
bronze_df = spark.read.format("delta").load("/mnt/bronze/nyctaxi_trips")

clean_df = bronze_df.select(
    "tpep_pickup_datetime", "tpep_dropoff_datetime",
    "trip_distance", "fare_amount", "pickup_zip", "dropoff_zip"
)

clean_df.write.format("delta") \
    .mode("overwrite") \
    .saveAsTable("main.nyctaxi.trips_cleaned")
```

📸 _Screenshot: Cleaned data preview or notebook cell_  
![Silver Layer Screenshot](https://chatgpt.com/g/g-p-680bf5618d8481918b06e8101d344bea-suraj/c/insert-image-here)

---

## 👓 Step 3: Create Stable View for Dashboard

- Created a **named view** `main.nyctaxi.trips_dashboard_view`
    
- Includes only safe, well-defined columns
    
- Used by production dashboards
    

📌 **Purpose:** Decouples dashboard from table schema, preventing breakage on evolution

```sql
CREATE OR REPLACE VIEW main.nyctaxi.trips_dashboard_view AS
SELECT
  tpep_pickup_datetime,
  tpep_dropoff_datetime,
  trip_distance,
  fare_amount,
  pickup_zip,
  dropoff_zip
FROM main.nyctaxi.trips_cleaned;
```

📸 _Screenshot: SQL view definition_  
![Stable View Screenshot](https://chatgpt.com/g/g-p-680bf5618d8481918b06e8101d344bea-suraj/c/insert-image-here)

---

## 🧪 Step 4: Create Latest View for Dev/Schema Exploration

- Created a dynamic view `main.nyctaxi.trips_dashboard_view_latest` with `SELECT *`
    
- Used by developers to explore new columns added via schema evolution
    

📌 **Purpose:** Safe experimentation without impacting production dashboards

```sql
CREATE OR REPLACE VIEW main.nyctaxi.trips_dashboard_view_latest AS
SELECT * FROM main.nyctaxi.trips_cleaned;
```

📸 _Screenshot: Dev view definition_  
![Dynamic View Screenshot](https://chatgpt.com/g/g-p-680bf5618d8481918b06e8101d344bea-suraj/c/insert-image-here)

---

## 🛡️ Step 5: Implement Schema Drift Detection

- Compared schema of Silver table vs. dashboard view
    
- Alerted if new fields were added or existing ones were removed
    

📌 **Purpose:** Early warning system for schema contract violations

```python
table_schema = set(f.name for f in spark.table("main.nyctaxi.trips_cleaned").schema)
view_schema = set(f.name for f in spark.table("main.nyctaxi.trips_dashboard_view").schema)

if table_schema != view_schema:
    print("⚠️ Schema drift detected!")
```

📸 _Screenshot: Output or drift detection logic_  
![Drift Detection Screenshot](https://chatgpt.com/g/g-p-680bf5618d8481918b06e8101d344bea-suraj/c/insert-image-here)

---

## 💡 Step 6: Author SQL Queries in Databricks SQL

Created key SQL queries using the view:

- **Fare by Pickup ZIP**
    
- **Avg Trip Distance by Dropoff ZIP**
    
- **Fare Over Time**
    

Example:

```sql
SELECT pickup_zip, SUM(fare_amount) AS total_fare
FROM main.nyctaxi.trips_dashboard_view
GROUP BY pickup_zip
ORDER BY total_fare DESC;
```

📸 _Screenshot: Query editor with saved query_  
![SQL Query Screenshot](https://chatgpt.com/g/g-p-680bf5618d8481918b06e8101d344bea-suraj/c/insert-image-here)

---

## 📊 Step 7: Create Visualizations

- Created visualizations using bar, line, and pie charts
    
- Linked them to saved queries
    

|Visualization Name|Type|Data Source|
|---|---|---|
|Fare by Pickup ZIP|Bar Chart|SQL Query 1|
|Avg Distance by Dropoff|Bar Chart|SQL Query 2|
|Daily Fare Trend|Line Chart|SQL Query 3|

📸 _Screenshot: Visualization configuration_  
![Visualization Screenshot](https://chatgpt.com/g/g-p-680bf5618d8481918b06e8101d344bea-suraj/c/insert-image-here)

---

## 🧩 Step 8: Assemble and Configure Dashboard

- Created `NYC Taxi Dashboard`
    
- Added visualizations, arranged layout
    
- Added **filters** for `pickup_zip`, `dropoff_zip`, and `pickup_date`
    

📌 **Purpose:** Enables interactive, dynamic exploration of trusted data

📸 _Screenshot: Dashboard layout_  
![Dashboard Screenshot](https://chatgpt.com/g/g-p-680bf5618d8481918b06e8101d344bea-suraj/c/insert-image-here)

---

## 🔐 Step 9: Access Control & Governance

- View and table were created under Unity Catalog
    
- Access managed via `GRANT` commands (UI or SQL)
    

📌 **Purpose:** Fine-grained governance for production data and dashboards

---

## ✅ Summary of Best Practices Implemented

|Layer|Best Practice|Applied?|
|---|---|---|
|Bronze|Schema evolution enabled|✅|
|Silver|Explicit schema projection|✅|
|Views|Stable + dynamic separation|✅|
|SQL|Modular queries|✅|
|Dash|Filters, visuals, governed views|✅|
|QA|Schema drift detection logic|✅|

---

Would you like this exported as HTML, Confluence Wiki format, or PDF? I can generate it in your preferred format.