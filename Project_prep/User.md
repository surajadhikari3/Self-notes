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

### ✅ Notes:

- Table points to data already saved.
    
- Separates **storage** and **metadata** steps.
    
- Used in distributed ingestion pipelines.
    

---

## 🔹 Summary Table

|Method|SQL/Python|Schema Definition|Table Type|Notes|
|---|---|---|---|---|
|`CREATE TABLE`|SQL|Explicit|Managed|Standard|
|CTAS|SQL|Inferred|Managed|Fast, simple|
|`.saveAsTable()`|PySpark|Inferred/manual|Managed|PySpark native|
|External Table|SQL|Explicit|Unmanaged|Path-based|
|Auto Loader|PySpark|Inferred|Managed|Streaming|
|Unity Catalog Table|SQL|Explicit|Managed/Unmanaged|Fine-grained access|
|Temporary Table|SQL|Inferred|Temp|Non-persistent|
|Register Existing Data|PySpark + SQL|Already saved|External|Two-step|

---

## 📎 Tips for Documentation

- Include architecture diagrams showing:  
    `Data Source ➝ Bronze ➝ Silver ➝ Gold ➝ BI Tools`
    
- Add links to data dictionary (column description, data type, nullable).
    
- Mention schema evolution strategies (`mergeSchema`, `overwriteSchema`).
    
- Include `COMMENT` statements to document tables and columns.
    

---

Would you like a **Markdown version**, **architecture diagram**, or **notebook template** with all of these documented for your internal docs?