
    

Databricks and Spark support a variety of real-time streaming patterns essential for modern data pipelines, analytics, and microservices.  
This document covers:

- Streaming design patterns
    
- Databricks usage (Auto Loader, CDC, etc.)
    

    

---

## **1️⃣ Simple Event Stream Processing**

### **Description:**

Process each incoming event in real time without aggregation.

### **Diagram:**

```
[Event Source] → [Spark Streaming Job] → [Sink]
```

### **Use Case:**

- IoT sensor readings
    
- Application logs
    

### **Databricks Example (Auto Loader)**

```python
df = (spark.readStream
      .format("cloudFiles")
      .option("cloudFiles.format", "json")
      .load("/mnt/events"))

df.writeStream.format("delta").option("checkpointLocation", "/mnt/checkpoint").start("/mnt/bronze")
```

---

## **2️⃣ Windowed Stream Processing**

### **Description:**

Aggregate events within **time windows** (tumbling, sliding, session).

### **Diagram:**

```
[Stream] → [Window Aggregation] → [Sink]
```

### **Use Case:**

- Per-minute sales stats
    
- Rolling average temperature
    

### **Databricks Example:**

```python
from pyspark.sql.functions import window, count

df.groupBy(window("timestamp", "10 minutes")).agg(count("*")).writeStream.format("console").start()
```

---

## **3️⃣ Stream-to-Table (Materialized View Pattern)**

### **Description:**

Keep a **stateful table updated in real time** from a stream.

### **Diagram:**

```
[Stream] → [State Store] → [Updatable Table]
```

### **Use Case:**

- Fraud detection with running totals
    
- User session management
    

### **Databricks Example (Delta Table Sink)**

```python
df.writeStream.outputMode("update").format("delta").option("checkpointLocation", "/mnt/chk").table("real_time_metrics")
```

---

## **4️⃣ Table-to-Stream (CDC Pattern)**

### **Description:**

Use **Change Data Capture (CDC)** to stream out changes from a database table.

### **Diagram:**

```
[Database] → [Debezium / CDC Connector] → [Kafka / Event Stream] → [Databricks Stream]
```

### **Use Case:**

- Sync on-prem DB changes to the cloud
    
- Event-driven microservices
    

### **Databricks Example (Kafka CDC Stream)**

```python
df = (spark.readStream.format("kafka")
      .option("kafka.bootstrap.servers", "broker:9092")
      .option("subscribe", "cdc_topic")
      .load())

df.selectExpr("CAST(value AS STRING)").writeStream.format("delta").start("/mnt/cdc_silver")
```

---

## **5️⃣ Stream-to-Stream Join**

### **Description:**

Join two live streams based on keys and time windows.

### **Diagram:**

```
[Stream A] ---+
               |-- [Join Logic] → [Sink]
[Stream B] ---+
```

### **Use Case:**

- Match payments with orders
    
- Real-time user profile enrichment
    

### **Databricks Example:**

```python
df1.join(df2, "orderId", "inner").writeStream.format("delta").start("/mnt/joined_data")
```

---

## **6️⃣ Stream Enrichment (Lookup Join)**

### **Description:**

Join streaming data with static or slowly changing reference data.

### **Diagram:**

```
[Stream] + [Lookup Table] → [Enriched Stream]
```

### **Use Case:**

- Add customer names to transaction streams
    

### **Databricks Example:**

```python
staticDF = spark.read.format("delta").table("customer_info")

streamingDF.join(staticDF, "customerId").writeStream.format("delta").start("/mnt/enriched")
```

---

## **7️⃣ Branching / Filtering Pattern**

### **Description:**

Split a stream into multiple branches based on conditions.

### **Diagram:**

```
[Stream] → [Branch 1: Fraud]  
        ↘ [Branch 2: Normal]
```

### **Use Case:**

- Fraud detection pipelines
    
- Routing messages to different sinks
    

### **Databricks Example:**

```python
fraudDF = df.filter("amount > 10000")
normalDF = df.filter("amount <= 10000")

fraudDF.writeStream.format("delta").start("/mnt/fraud")
normalDF.writeStream.format("delta").start("/mnt/normal")
```

---

## **8️⃣ Dead Letter Stream (Error Handling Pattern)**

### **Description:**

Capture problematic events separately for reprocessing.

### **Diagram:**

```
[Stream] → [Process]  
           ↘ [Dead Letter Queue]
```

### **Use Case:**

- Handle parse failures gracefully
    

### **Databricks Example:**

```python
def safe_parse(row):
    try:
        return parse_event(row)
    except Exception:
        return {"error": row}

parsedDF = df.rdd.map(safe_parse).toDF()
```

---

## **9️⃣ Outbox Pattern**

### **Description:**

Write to DB & message queue (Kafka) in a single transaction via an **Outbox table**.

### **Diagram:**

```
[Service] → [DB + Outbox Table] → [CDC to Kafka] → [Downstream Consumers]
```

### **Use Case:**

- Ensure event delivery in microservices
    
- Avoid dual-write problems
    

---

## **🔟 Streaming Aggregation**

### **Description:**

Real-time metrics like **sum, count, avg** over event streams.

### **Diagram:**

```
[Stream] → [Aggregator] → [Dashboard/Table]
```

### **Databricks Example:**

```python
df.groupBy("category").count().writeStream.format("delta").start("/mnt/category_counts")
```

---

## **1️⃣1️⃣ Fan-in / Fan-out**

### **Description:**

- **Fan-in:** Merge multiple streams
    
- **Fan-out:** Publish to multiple sinks
    

### **Diagram:**

```
Fan-in:  [Stream1] + [Stream2] → [Unified Stream]  
Fan-out: [Stream] → [Sink1 + Sink2 + Sink3]
```

---

## **1️⃣2️⃣ Replay / Rewind Pattern**

### **Description:**

Reprocess historical data by reading from persisted storage.

### **Diagram:**

```
[Delta Table / Kafka Retention] → [Reprocess Job]
```

### **Use Case:**

- Fix pipeline bugs
    
- Backfill data
    

---

# **Databricks-Specific Tools Involved**

|Feature|Pattern|
|---|---|
|**Auto Loader**|Simple Stream Ingestion|
|**Delta Lake**|Stream-to-Table, Replay|
|**Unity Catalog**|Governance over Streaming Tables|
|**Debezium/Kafka + Databricks**|CDC (Table-to-Stream)|
|**Structured Streaming API**|All stream transformations|

---

# **Summary Table**

|Pattern|Use Case|
|---|---|
|Simple Stream|Real-time events|
|Windowed Aggregation|Time-based metrics|
|Stream-to-Table|Stateful pipelines|
|Table-to-Stream (CDC)|DB change capture|
|Stream Join|Enrichment|
|Branching|Routing data|
|Dead Letter Stream|Error capture|
|Outbox Pattern|Microservice event delivery|
|Replay/Rewind|Backfill pipelines|

---

# **Conclusion**

These patterns help build **robust, scalable real-time systems** in Databricks using Spark Structured Streaming.

---

### **Next Steps:**

- Use this document directly in **Confluence**
    
- For diagrams in visual form, I can help you generate **Excalidraw, Mermaid, or Lucidchart diagrams**
    
- Would you like me to export these as Markdown, Confluence Cloud Macro, or an image diagram?
    

![[Pasted image 20250721121945.png]]