Workspace 

Shared Vs Users

### **Users Workspace (`/Users`)**

- **Path**: `/Users/<email>`
    
- **Purpose**: Personal, private development space for individual users.
    
- **Access**:
    
    - Only accessible by the user who owns it (unless permissions are explicitly granted).
        
    - Great for experimentation, prototyping, testing, or learning.
        

#### ✅ Typical Use Cases:

- Writing personal notebooks
    
- Running test jobs
    
- Drafting pipelines or ETL logic
    

---

### 🤝 **2. Shared Workspace (`/Shared`)**

- **Path**: `/Shared`
    
- **Purpose**: Collaboration space shared across multiple users and teams.
    
- **Access**:
    
    - Anyone with access to the Databricks workspace can be given permission to view or edit resources in `/Shared`.
        
    - Permissions can be configured at the folder or notebook level.
        

#### ✅ Typical Use Cases:

- Production-ready notebooks
    
- Shared dashboards or reports
    
- Team notebooks or documentation
    
- Scheduled workflows or shared utilities
    

---

###  **Access Control Comparison**

|Feature|`/Users` Workspace|`/Shared` Workspace|
|---|---|---|
|Default visibility|Private to owner|Visible to team (if shared)|
|Collaboration|No (by default)|Yes|
|Access Control Supported?|Yes|Yes|
|Best For|Individual development|Team collaboration|

---

###  Recommendation

- **Use `/Users/email`** for experimentation and POCs.
    
- **Move to `/Shared`** once code or notebooks need to be reviewed, used in a job, or shared with teammates.

Managed Vs Foreign Catalog:


|Category|Managed Catalog|Foreign Catalog|
|---|---|---|
|Ownership|Databricks-managed|External system-managed|
|Source Type|Internal (Delta Lake, Unity Catalog)|External systems (e.g., PostgreSQL, MySQL, Oracle, Glue)|
|Storage Location|DBFS or external cloud storage linked via Unity Catalog|Resides in the external system; not copied to Databricks|
|Data Governance|Fully governed by Unity Catalog|Governance remains with external source|
|Read/Write Capability|Read and write supported|Mostly read-only|
|Table Metadata|Managed by Unity Catalog|Virtualized from external source|
|Access Control|Fine-grained (catalog, schema, table, column, row)|Catalog-level only|
|Audit Logging|Supported via Unity Catalog system tables|Not supported|
|Data Lineage|Tracked automatically|Not available|
|Delta Sharing|Supported|Not supported|
|Performance Optimization|Uses Delta caching and execution plans|Depends on source system performance|
|Schema Evolution|Supported|Not applicable|
|External Location Support|Supported|Not applicable|
|Tagging and Classification|Supported|Not supported|
|Use Case|New lakehouse implementations with full governance needs|Querying data from legacy or federated external sources|
|Best For|Secure, governed analytics on Delta Lake|Ad-hoc queries, BI access, or phased migration scenarios|
|Data Federation|Not required|Required (via Lakehouse Federation)|
|Integration Type|Native to Unity Catalog|JDBC-based or cloud-native federation|
|Migration Path|End-state target|Temporary bridge or hybrid access strategy|


Here's a **clean, clear, and precise documentation-style answer** to the question:

---

# **How to Get Data from External SQL Server to Databricks**

---

Here is a **well-documented guide** on all the schema creation options in **Databricks**, including SQL, PySpark, Unity Catalog, CTAS, Autoloader, streaming, and external tables — all backed with code examples and notes for documentation or training purposes.

---

# 📘 Databricks Table Schema Creation – Complete Guide

---

## 🔹 1. **Create Table with Explicit Schema (SQL)**

```sql
CREATE TABLE catalog_name.schema_name.table_name (
    id INT,
    name STRING,
    signup_date DATE
);
```

### ✅ Notes:

- Best for manual control.
    
- Table is **managed** (Databricks controls storage location).
    
- Works for Unity Catalog or Hive Metastore.
    

---

## 🔹 2. **Create External Table with Schema + Location**

```sql
CREATE TABLE catalog_name.schema_name.table_name (
    id INT,
    name STRING
)
USING DELTA
LOCATION 'abfss://container@account.dfs.core.windows.net/path/to/folder/';
```

### ✅ Notes:

- You define storage (`abfss`, `s3`, etc.).
    
- Metadata stored in metastore.
    
- Table is **unmanaged**.
    

---

## 🔹 3. **CTAS – Create Table As Select (Schema is inferred)**

```sql
CREATE TABLE catalog_name.schema_name.active_users
USING DELTA
AS
SELECT id, name FROM raw_data.users WHERE status = 'active';
```

### ✅ Notes:

- **No need to define schema manually**.
    
- Schema is taken from the `SELECT` query result.
    
- Supports formats: `DELTA`, `PARQUET`, `CSV`, etc.
    

---

## 🔹 4. **Create Table from DataFrame (PySpark)**

```python
df = spark.read.json("/mnt/raw/users.json")

df.write.format("delta").saveAsTable("catalog.schema.users")
```

### ✅ Notes:

- You can let Spark infer schema from file.
    
- Use `.saveAsTable()` for managed tables.
    
- Alternative: `.save(path)` + `CREATE TABLE ... LOCATION`.
    

---

## 🔹 5. **Define Schema in PySpark Programmatically**

```python
from pyspark.sql.types import StructType, StructField, StringType, IntegerType

schema = StructType([
    StructField("id", IntegerType(), True),
    StructField("name", StringType(), True)
])

df = spark.read.schema(schema).json("/mnt/raw/users.json")

df.write.format("delta").saveAsTable("catalog.schema.defined_users")
```

### ✅ Notes:

- Manual schema avoids issues with incorrect inference.
    
- Helpful for production pipelines.
    

---

## 🔹 6. **Streaming Table Creation via Auto Loader (Schema inferred)**

```python
df = (
  spark.readStream.format("cloudFiles")
  .option("cloudFiles.format", "json")
  .load("abfss://stream@lakehouse.dfs.core.windows.net/incoming/")
)

df.writeStream.format("delta") \
  .option("checkpointLocation", "/mnt/checkpoints/incoming/") \
  .table("bronze_layer.incoming_events")
```

### ✅ Notes:

- Schema inferred from incoming JSON files.
    
- Table automatically created (if not exists).
    
- Recommended to use `schemaLocation` and enable schema evolution.
    

---

## 🔹 7. **Create Table in Unity Catalog (UC)**

```sql
CREATE TABLE main.analytics_db.transactions (
    txn_id STRING,
    amount DOUBLE,
    timestamp TIMESTAMP
)
USING DELTA
COMMENT "Financial transaction data"
LOCATION 'abfss://bronze@datalake.dfs.core.windows.net/transactions/';
```

### ✅ Notes:

- `main` = Unity Catalog root.
    
- Recommended for **centralized governance**, **lineage**, **RBAC**.
    
- Supports managed and external tables.
    

---

## 🔹 8. **Create Table from Views**

```sql
CREATE OR REPLACE TEMP VIEW temp_view AS
SELECT id, name FROM users;

CREATE TABLE new_users_table
AS SELECT * FROM temp_view;
```

### ✅ Notes:

- Views can act as intermediates in pipeline.
    
- Used often in transformation pipelines before creating Gold layer.
    

---

## 🔹 9. **Create Temporary Table (Session-Scope Only)**

```sql
CREATE TEMPORARY VIEW temp_users AS
SELECT * FROM raw_data.users;
```

### ✅ Notes:

- Used for intermediate logic in a notebook session.
    
- Doesn’t persist between sessions.
    

---

## 🔹 10. **Register Existing Data Folder as Table**

```python
df.write.format("delta").save("abfss://zone@lakehouse.dfs.core.windows.net/gold/clean_data/")

spark.sql("""
  CREATE TABLE catalog.schema.clean_data
  USING DELTA
  LOCATION 'abfss://zone@lakehouse.dfs.core.windows.net/gold/clean_data/'
""")
```

### **Steps:**

1. **Choose Data Source:**
    
    - Click **“Upload File”** or select from existing mount points (e.g., `/mnt/data/`, DBFS, ADLS, S3).
        
    - Optionally drag and drop your file.
        
2. **Select File Format:**
    
    - Supported: CSV, JSON, Parquet, Avro, ORC, Delta.
        
    - The file is **previewed with automatic schema detection**.
        
3. **Configure Table Settings:**
    
    - Choose:
        
        - **Catalog** (e.g., `main`)
            
        - **Schema** (aka Database, e.g., `bronze_layer`)
            
        - **Table Name**
            
        - **Table Type**:
            
            - `Managed Table` (default)
                
            - `External Table` (optional if you specify a location)
                
4. **Edit Schema (Optional):**
    
    - You can rename columns, change data types, and set nullability.
        
5. **Choose Table Format:**
    
    - `DELTA` (recommended)
        
    - `PARQUET`, `CSV`, etc.
        
6. **Advanced Options (Optional):**
    
    - Set custom table comment
        
    - Modify `LOCATION` path if creating an external table
        
7. **Click `Create Table`:**
    
    - Table is created and registered in metastore or Unity Catalog.
        
    - Optionally click **“Create notebook”** for immediate exploration.

### ✅ Notes:

- Table points to data already saved.
    
- Separates **storage** and **metadata** steps.
    
- Used in distributed ingestion pipelines.
    

---

## 🔹 Summary Table

| Method                 | SQL/Python    | Schema Definition | Table Type        | Notes               |
| ---------------------- | ------------- | ----------------- | ----------------- | ------------------- |
| `CREATE TABLE`         | SQL           | Explicit          | Managed           | Standard            |
| CTAS                   | SQL           | Inferred          | Managed           | Fast, simple        |
| `.saveAsTable()`       | PySpark       | Inferred/manual   | Managed           | PySpark native      |
| External Table         | SQL           | Explicit          | Unmanaged         | Path-based          |
| Auto Loader            | PySpark       | Inferred          | Managed           | Streaming           |
| Unity Catalog Table    | SQL           | Explicit          | Managed/Unmanaged | Fine-grained access |
| Temporary Table        | SQL           | Inferred          | Temp              | Non-persistent      |
| Register Existing Data | PySpark + SQL | Already saved     | External          | Two-step            |

---

## 📎 Tips for Documentation

- Include architecture diagrams showing:  
    `Data Source ➝ Bronze ➝ Silver ➝ Gold ➝ BI Tools`
    
- Add links to data dictionary (column description, data type, nullable).
    
- Mention schema evolution strategies (`mergeSchema`, `overwriteSchema`).
    
- Include `COMMENT` statements to document tables and columns.
    

---


In **Apache Spark**, the **`overwrite`** and **`append`** modes are used during write operations (e.g., when saving a DataFrame to a file system, table, or Delta Lake). Each serves a different purpose depending on your data processing scenario.


##  `append` vs `overwrite` in Spark Write Operations

|Mode|Description|Use Case|Caution|
|---|---|---|---|
|**`append`**|Adds new data to the existing dataset without modifying the previous data|- Streaming pipelines- Historical data logging- Adding new daily/hourly data|Can lead to duplicates if not de-duplicated beforehand|
|**`overwrite`**|Replaces the existing dataset completely with the new data|- Batch pipelines where old data should be replaced- Full refresh scenarios|**Deletes existing data**, may cause data loss if not careful|

---

## 📘 Code Examples

### 1. **Append Mode**

```python
df.write.mode("append").parquet("/path/to/output")
```

- Will **add new data** to the folder (or table) at `/path/to/output`.
    
- Useful in **streaming**, **event log**, or **incremental batch** jobs.
    

### 2. **Overwrite Mode**

```python
df.write.mode("overwrite").parquet("/path/to/output")
```

- Will **remove all existing data** in `/path/to/output` and replace it with new data.
    
- Use it when you're doing **full data refresh** or **snapshot ingestion**.
    

---

## 🛠️ Delta Lake Specific Behavior (Important!)

If using **Delta Lake**, overwrite mode can be **partition-aware**:

```python
df.write.mode("overwrite").option("overwriteSchema", "true").partitionBy("date").format("delta").save("/delta/events")
```

And you can **overwrite specific partitions**:

```python
df.write \
  .mode("overwrite") \
  .option("replaceWhere", "date = '2024-01-01'") \
  .format("delta") \
  .save("/delta/events")
```

---

##  When to Use What?

|Scenario|Recommended Mode|
|---|---|
|Real-time or micro-batch ingestion|`append`|
|Historical data refresh (e.g., daily snapshot)|`overwrite`|
|Upsert or update specific partitions|`overwrite` with `replaceWhere` (Delta only)|
|Caution: frequent overwrite on the same path|Can cause **concurrent write conflicts** or **performance degradation**|

Great question! In Delta Lake (and Spark in general), when we say **“partition-aware”**, we mean that **write or read operations understand and utilize the underlying table's partition structure** to be **more efficient** and **safe**.

---

## 📦 What is a Partition?

Partitioning is a technique to **logically divide large datasets** into smaller chunks based on a column (or columns) — like `date`, `region`, or `category`.

Example:

```python
df.write.partitionBy("date").format("delta").save("/delta/sales")
```

This will create folders like:

```
/delta/sales/date=2024-01-01/
/delta/sales/date=2024-01-02/
...
```

---

## 🔍 What Does “Partition-Aware” Mean?

When a Spark/Delta **write operation is partition-aware**, it means:

|Feature|Behavior|
|---|---|
|✅ **Targeted overwrite**|Only the affected partition is replaced, not the whole table|
|✅ **Efficient read/write**|Spark reads/writes only relevant partitions|
|✅ **Safe schema enforcement**|Prevents accidental data loss across other partitions|
|✅ **Optimized performance**|Less I/O and shuffling during jobs|

---

## 🧪 Example: Overwriting Specific Partition

Instead of overwriting the whole dataset:

```python
df.write.mode("overwrite").format("delta").save("/delta/sales")  # DANGEROUS
```

You can safely do **partition-aware overwrite**:

```python
df.write \
  .format("delta") \
  .mode("overwrite") \
  .option("replaceWhere", "date = '2024-01-01'") \
  .save("/delta/sales")   # SAFE and targeted
```

💡 Spark will only delete and rewrite the partition where `date = '2024-01-01'`.

---

## ⚠️ Without Partition Awareness

If you don't use `replaceWhere` or if the table isn't partitioned:

- An overwrite replaces **everything** — all files, even for dates not related to the current batch.
    
- Performance and data integrity suffer.
    

---

## 🧠 Layman Analogy

Imagine your data table is a **filing cabinet**, partitioned by **date folders**.

- **Overwrite full table** = You throw away the **whole cabinet**.
    
- **Partition-aware overwrite** = You **replace only one folder (like Jan 1)**, keeping the rest untouched.
    

---

Let me know if you'd like a **visual partition-aware vs unaware write comparison** — I can generate one too.





## All Write Modes in Delta Lake

|Mode|Description|Best Used For|
|---|---|---|
|`append`|Adds new rows to the table|Streaming, event logs|
|`overwrite`|Replaces **entire dataset**|Full snapshot replacements|
|`overwrite + replaceWhere`|Replaces only matching partitions|Partition-aware updates|
|`ignore`|Writes **only if the table doesn’t exist**|Initial loads|
|`error` or `errorifexists`|Fails if table/path already exists|Prevents accidental overwrite|