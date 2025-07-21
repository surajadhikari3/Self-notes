
# **📄 Databricks Streaming Patterns with File-Based Sources**

## **Overview**

Databricks supports various **file-based streaming ingestion patterns** using **Auto Loader**, **Delta Lake**, and **Spark Structured Streaming**.  
These patterns are ideal for ingesting data from:

- Cloud Storage (S3, ADLS, GCS)
    
- On-prem batch files (CSV, JSON, Parquet, etc.)
    
- CDC events written as files
    

---

## **1️⃣ Auto Loader Pattern (File Ingestion Stream)**

### **Description:**

Auto Loader incrementally ingests new files from cloud storage into Databricks without managing file state manually.

### **Diagram:**

```
[Cloud Storage] → [Auto Loader] → [Bronze Delta Table]
```

### **Use Case:**

- New files landing in a folder (IoT, logs, batch drops)
    
- Cost-efficient ingestion for semi-structured data
    

### **Example:**

```python
df = (spark.readStream.format("cloudFiles")
      .option("cloudFiles.format", "json")
      .load("/mnt/raw-data/"))

df.writeStream.format("delta").option("checkpointLocation", "/mnt/checkpoint").start("/mnt/bronze")
```

---

## **2️⃣ Bronze-Silver-Gold Pipeline Pattern (Medallion Architecture)**

### **Description:**

Process raw files to clean data (Silver) and business aggregates (Gold).

### **Diagram:**

```
[File Source] → [Bronze (Raw)] → [Silver (Cleaned)] → [Gold (Aggregated)]
```

### **Use Case:**

- Data lakehouse ETL pipelines
    
- Analytical dashboards
    

### **Example:**

```python
bronzeDF = spark.readStream.format("delta").table("bronze_table")

silverDF = bronzeDF.filter("is_valid = true")

silverDF.writeStream.format("delta").table("silver_table")
```

---

## **3️⃣ File-based CDC Pattern (Incremental Change Capture)**

### **Description:**

Capture database changes exported as files (CSV/JSON) and process them in Databricks.

### **Diagram:**

```
[DB Change Logs] → [File Drop] → [Auto Loader] → [CDC Processing]
```

### **Use Case:**

- On-prem DB sync using exported change files
    
- Hybrid cloud migrations
    

### **Example:**

```python
cdcDF = (spark.readStream.format("cloudFiles")
         .option("cloudFiles.format", "json")
         .load("/mnt/cdc-events/"))

cdcDF.writeStream.format("delta").table("cdc_bronze")
```

---

## **4️⃣ Streaming Join with File Source**

### **Description:**

Join real-time file streams with static datasets (customer info, product lookup).

### **Diagram:**

```
[File Stream] + [Static Table] → [Enriched Stream]
```

### **Use Case:**

- Add metadata or dimensions to raw streams
    

### **Example:**

```python
customerDF = spark.read.format("delta").table("customers")

streamDF.join(customerDF, "customer_id").writeStream.format("delta").start("/mnt/enriched")
```

---

## **5️⃣ Streaming File Aggregation (Windowed Pattern)**

### **Description:**

Aggregate file-based events into time windows (e.g., hourly totals).

### **Diagram:**

```
[File Stream] → [Window Aggregation] → [Delta Table]
```

### **Use Case:**

- Real-time monitoring dashboards
    
- Business KPI updates
    

### **Example:**

```python
from pyspark.sql.functions import window, count

streamDF.groupBy(window("event_time", "1 hour")).agg(count("*")).writeStream.format("delta").start("/mnt/hourly_metrics")
```

---

## **6️⃣ Dead Letter Folder Pattern (Error Isolation)**

### **Description:**

Store bad or corrupt records into a separate file folder for inspection.

### **Diagram:**

```
[File Stream] → [Parse]  
               ↘ [Dead Letter Folder]
```

### **Use Case:**

- Handle schema drift
    
- Capture corrupt records
    

---

## **7️⃣ Replay & Reprocess Pattern**

### **Description:**

Re-read files from Delta tables for backfill or bug fixes.

### **Diagram:**

```
[Delta Table] → [Reprocessing Job]
```

### **Use Case:**

- Historical data replay
    
- Fixing pipeline logic issues
    

---

## **Summary Table**

|Pattern|Description|Use Case|
|---|---|---|
|Auto Loader|Incremental file ingestion|IoT, batch files|
|Bronze-Silver-Gold|Layered data refinement|ETL pipelines|
|File-based CDC|Capture DB changes via files|Hybrid cloud sync|
|Stream Join|Enrich data streams|Metadata joins|
|Window Aggregation|Group events by time|KPI dashboards|
|Dead Letter Folder|Handle bad data|Error isolation|
|Replay/Reprocess|Historical re-run|Bug fixes|

---

## **Databricks Tools Involved**

|Tool|Purpose|
|---|---|
|**Auto Loader**|Efficient file ingestion|
|**Delta Lake**|ACID storage for streaming|
|**Unity Catalog**|Access governance|
|**Structured Streaming API**|Real-time pipelines|

---
