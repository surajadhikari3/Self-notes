
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