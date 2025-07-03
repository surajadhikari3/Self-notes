
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

---

Let me know if you'd like a Markdown or HTML version of this for direct inclusion in your documentation!