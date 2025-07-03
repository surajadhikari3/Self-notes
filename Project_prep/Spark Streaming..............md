
Auto Loader (Supports Volume/DBFS/ Cloud Storage Objects)............

**Auto Loader** is Databricks' **incremental file ingestion engine** for streaming large volumes of files efficiently, using:

- **Directory listing** (for DBFS/Volumes)
    
- **File notification services** (for cloud storage like S3/ADLS)


## POC Objective:

Demonstrate how Apache Spark Structured Streaming processes real-time data from a file system (Volumes) using:

- **Bronze → Silver → Gold** architecture
    
- **Stream-static joins**
    
- **Watermarking and window-based aggregations**
    
- **Delta Lake** for transactional streaming
    
- **Auto Loader** 
    
- Output written as **queryable Delta tables**
    

---

Components to Include

### 1. **Raw Input Simulation**

- 2 JSON files dropped as micro-batches     

### 2. **Bronze Layer** – Raw Ingestion

- Uses  `cloudFiles`
    
- Writes to a Delta location with checkpointing
    
- Registered as `bronze_events` table
    

### 3. **Silver Layer** – Enrichment via Stream-Static Join

- Joins Bronze with a static `country_codes` Delta table
    
- Creates `silver_events` table
    

### 4. **Gold Layer** – Analytics with Watermarking

- Groups data by country + event_type in 5-minute windows
    
- Applies watermark of 10 minutes to handle late data
    
- Writes to `gold_aggregated_events` table
    

---
Key Streaming Patterns Highlighted

| Pattern                       | Where it's shown                         |
| ----------------------------- | ---------------------------------------- |
| **Raw file stream ingestion** | Bronze layer                             |
| **Stream-static join**        | Silver layer with country lookup         |
| **Watermarking**              | Gold layer to handle late data           |
| **Window aggregation**        | Gold layer (5-min rolling window)        |
| **Delta Lake storage**        | All layers use Delta tables              |
| **Checkpoints**               | Enabled per layer for fault tolerance    |
| **Schema enforcement**        | JSON with defined schema in Bronze layer |
Absolutely! Here's a solid list of **Spark Structured Streaming patterns specific to file-based systems** — ideal for including in **POC documentation** or technical architecture references.

---

## 📄 Spark Structured Streaming Patterns (File System Ingestion)

---

### 🔷 1. **Basic File-Based ETL Pattern**

**Pattern:** Monitor a folder for new files (e.g., JSON), filter and write to Delta  
**Use Case:** Ingest structured logs or CSVs into Delta Lake

```python
df = (spark.readStream
      .format("json")
      .schema(schema)
      .load("/data/input/"))

df.filter("status = 'active'") \
  .writeStream.format("delta") \
  .outputMode("append") \
  .start("/data/output/")
```

---

### 🔷 2. **Auto Loader-Based Ingestion Pattern**

**Pattern:** Use Auto Loader for scalable and schema-aware ingestion  
**Use Case:** High-volume, append-only file ingestion from cloud/Volumes

```python
df = (spark.readStream
      .format("cloudFiles")
      .option("cloudFiles.format", "json")
      .option("cloudFiles.schemaLocation", "/data/schema/")
      .load("/data/input/"))
```

---

### 🔷 3. **Stream-Static Join Pattern**

**Pattern:** Enrich streaming records with lookup data  
**Use Case:** Add product, country, or user details from a static dimension table

```python
stream_df.join(static_df, "country_code")
```

---

### 🔷 4. **Stream-Stream Join Pattern**

**Pattern:** Join two live streams based on event time and keys  
**Use Case:** Correlate user actions with transactions

```python
stream1.withWatermark("time", "10 minutes")
  .join(stream2.withWatermark("time", "10 minutes"), "user_id")
```

---

### 🔷 5. **Deduplication with Watermark**

**Pattern:** Remove duplicate events using `dropDuplicates` and watermark  
**Use Case:** Avoid processing the same file or event twice

```python
df.withWatermark("event_time", "10 minutes")
  .dropDuplicates(["event_id", "event_time"])
```

---

### 🔷 6. **Windowed Aggregation Pattern**

**Pattern:** Group events into tumbling or sliding time windows  
**Use Case:** Count events per 5-minute window per event type

```python
df.withWatermark("event_time", "10 minutes") \
  .groupBy(window("event_time", "5 minutes"), "event_type") \
  .count()
```

---

### 🔷 7. **Multi-Sink Fanout Pattern**

**Pattern:** Write same stream to multiple destinations  
**Use Case:** Persist in Delta and simultaneously push to monitoring

```python
def multi_sink(batch_df, _):
    batch_df.write.format("delta").save("/data/output")
    batch_df.write.format("console").save()

df.writeStream.foreachBatch(multi_sink).start()
```

---

### 🔷 8. **Bronze → Silver → Gold Pattern**

**Pattern:** Layered streaming architecture for separation of concerns  
**Use Case:** Scalable, modular data lake with raw → refined → aggregated flows

```plaintext
Raw Files → Bronze (Delta) → Enriched Silver → Aggregated Gold
```

---

### 🔷 9. **Dead-Letter Queue Pattern**

**Pattern:** Capture malformed or corrupt records to a separate Delta path  
**Use Case:** Prevent stream failure due to bad input data

```python
try:
    valid = df.select(from_json(col("raw"), schema))
except:
    df.writeStream.format("delta").start("/dead-letter/")
```

---

### 🔷 10. **Schema Evolution Handling (Auto Loader)**

**Pattern:** Automatically handle new columns added to incoming files  
**Use Case:** Evolving upstream schema (e.g., logs or sensors)

```python
.option("mergeSchema", "true")
```

---

## ✅ Summary Table

|Pattern|Benefit|
|---|---|
|Basic File ETL|Simple raw-to-clean transformation|
|Auto Loader Ingestion|Scalable and schema-aware|
|Stream-Static Join|Adds context to records|
|Stream-Stream Join|Correlates events across sources|
|Watermark + Deduplication|Safe late-data handling + no duplicates|
|Window Aggregation|Time-based insights|
|Multi-Sink Output|Real-time + batch storage|
|Bronze → Silver → Gold|Modular & maintainable pipeline|
|Dead-Letter Queue|Resilience to bad data|
|Auto Schema Evolution|Adapts to incoming structure changes|

--------------------------------

Absolutely! Here's a clean, beginner-friendly and **fully explained Spark Structured Streaming POC** using **two streaming sources**, **Auto Loader**, and includes:

- ✅ Deduplication with watermark
    
- ✅ Stream-to-stream join
    
- ✅ Windowed aggregation
    
- ✅ Bronze → Silver → Gold layers
    
- ✅ Delta table registration
    

No complications — just what you've asked for.

---

## 📁 Assumptions

You're using this path structure:

```python
base_path = "/Volumes/main/default/stream_demo"
```

|Component|Path|
|---|---|
|Stream 1 Files|`/Volumes/main/default/stream_demo/input_1/`|
|Stream 2 Files|`/Volumes/main/default/stream_demo/input_2/`|
|Static Country|`/Volumes/main/default/stream_demo/country_codes.csv`|

---

## ✅ Step-by-Step Code and Explanation

---

### 🟫 1. Bronze Layer – Read Both Streams Using Auto Loader with Deduplication

#### Stream 1 (user events)

```python
stream1_bronze = (
  spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "json")
    .option("cloudFiles.schemaLocation", f"{base_path}/_schema/stream1")
    .load(f"{base_path}/input_1")
    .withWatermark("event_time", "10 minutes")               # allow late data
    .dropDuplicates(["event_id", "event_time"])              # prevent reprocessing same event
)

stream1_bronze.writeStream \
  .format("delta") \
  .outputMode("append") \
  .option("checkpointLocation", f"{base_path}/checkpoints/bronze/stream1") \
  .start(f"{base_path}/bronze_events_1")
```

---

#### Stream 2 (transactions)

```python
stream2_bronze = (
  spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "json")
    .option("cloudFiles.schemaLocation", f"{base_path}/_schema/stream2")
    .load(f"{base_path}/input_2")
    .withWatermark("event_time", "10 minutes")
    .dropDuplicates(["event_id", "event_time"])
)

stream2_bronze.writeStream \
  .format("delta") \
  .outputMode("append") \
  .option("checkpointLocation", f"{base_path}/checkpoints/bronze/stream2") \
  .start(f"{base_path}/bronze_events_2")
```

---

### 🥈 2. Silver Layer – Enrich and Join Streams

#### Read both bronze layers

```python
bronze_df_1 = spark.readStream.format("delta").load(f"{base_path}/bronze_events_1")
bronze_df_2 = spark.readStream.format("delta").load(f"{base_path}/bronze_events_2")
```

#### Read static country table (already saved as Delta table)

```python
country_df = spark.read.format("delta").load(f"{base_path}/country_codes_delta")
```

#### Enrich Stream 1 with country name

```python
bronze_df_1 = bronze_df_1.join(country_df, on="country_code", how="left")
```

#### Stream-to-Stream Join

```python
from pyspark.sql.functions import expr

joined_df = bronze_df_1 \
  .withWatermark("event_time", "10 minutes") \
  .join(
    bronze_df_2.withWatermark("event_time", "10 minutes"),
    expr("""
      bronze_df_1.user_id = bronze_df_2.user_id AND
      bronze_df_1.event_time BETWEEN bronze_df_2.event_time - INTERVAL 5 minutes AND bronze_df_2.event_time + INTERVAL 5 minutes
    """)
  )
```

#### Write Silver Layer

```python
joined_df.writeStream \
  .format("delta") \
  .outputMode("append") \
  .option("checkpointLocation", f"{base_path}/checkpoints/silver") \
  .start(f"{base_path}/silver_events")
```

---

### 🥇 3. Gold Layer – Windowed Aggregation

```python
from pyspark.sql.functions import window

silver_df = spark.readStream.format("delta").load(f"{base_path}/silver_events")

gold_df = silver_df \
  .withWatermark("event_time", "10 minutes") \
  .groupBy(
    window("event_time", "5 minutes"),
    "country_name",
    "event_type"
  ).count()
```

#### Write to Delta

```python
gold_df.writeStream \
  .format("delta") \
  .outputMode("append") \
  .option("checkpointLocation", f"{base_path}/checkpoints/gold") \
  .start(f"{base_path}/gold_aggregated")
```

---

### 📦 4. Register All Delta Tables for SQL Access

```python
# Bronze Tables
spark.read.format("delta").load(f"{base_path}/bronze_events_1") \
  .write.mode("overwrite").saveAsTable("main.default.bronze_events_1")

spark.read.format("delta").load(f"{base_path}/bronze_events_2") \
  .write.mode("overwrite").saveAsTable("main.default.bronze_events_2")

# Silver Table
spark.read.format("delta").load(f"{base_path}/silver_events") \
  .write.mode("overwrite").saveAsTable("main.default.silver_events")

# Gold Table
spark.read.format("delta").load(f"{base_path}/gold_aggregated") \
  .write.mode("overwrite").saveAsTable("main.default.gold_aggregated")
```

---

## ✅ Summary of What This POC Shows

|Pattern|Where Used|
|---|---|
|**Auto Loader**|For both streaming sources (input_1 & 2)|
|**Deduplication + Watermark**|Bronze layer with `dropDuplicates`|
|**Stream-Static Join**|Bronze stream 1 + country table|
|**Stream-Stream Join**|Silver layer — on user_id & time window|
|**Windowed Aggregation**|Gold layer — 5-minute event summaries|
|**Delta Lake Output**|All layers use Delta|
|**SQL-accessible Tables**|`saveAsTable` registered output|

---

Let me know if you want this wrapped into a `.ipynb` file or want to add alerts or dashboards to visualize the gold layer!