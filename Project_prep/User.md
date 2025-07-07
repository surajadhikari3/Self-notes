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

## 1. **Spark JDBC Connector (Direct Ingestion)**

### **Use Case**: Batch ingestion of SQL Server data into Databricks-managed Delta Lake tables.

### **Description**:

Databricks supports reading from SQL Server using the built-in Spark JDBC connector. It allows direct querying and ingestion of SQL Server tables or queries into Spark DataFrames, which can then be written to Delta Lake.

### **Steps**:

```python
jdbc_url = "jdbc:sqlserver://<server>:1433;database=<db>"
connection_properties = {
    "user": "<username>",
    "password": "<password>",
    "driver": "com.microsoft.sqlserver.jdbc.SQLServerDriver"
}

df = spark.read.jdbc(
    url=jdbc_url,
    table="dbo.my_table",
    properties=connection_properties
)

df.write.format("delta").mode("overwrite").save("/mnt/bronze/my_table")
```

### **Key Features**:

- Supports custom SQL queries with `dbtable` as a subquery
    
- Can be automated with jobs or workflows
    
- No external staging required
    

---

## 2. **Autoloader + Blob Storage (Staged Ingestion Pipeline)**

### **Use Case**: Scalable, schema-aware ingestion using file-based export from SQL Server.

### **Description**:

Data is first exported from SQL Server into a **staging area** (e.g., Azure Blob Storage) in a file format like CSV or Parquet. Databricks **Autoloader** then watches the storage location and incrementally ingests new files into Delta Lake.

### **Steps**:

1. Use SSIS, Python, or a script to export SQL Server tables as files to Blob Storage.
    
2. Use Autoloader to continuously ingest those files:
    

```python
df = spark.readStream.format("cloudFiles") \
    .option("cloudFiles.format", "parquet") \
    .load("abfss://staging@<storage_account>.dfs.core.windows.net/sqlserver/")

df.writeStream.format("delta") \
    .option("checkpointLocation", "/mnt/checkpoints/my_table") \
    .start("/mnt/bronze/my_table")
```

### **Key Features**:

- Supports schema inference and evolution
    
- Highly scalable and efficient for micro-batch ingestion
    
- Great for CDC-based ingestion pipelines
    

---

## 3. **Azure Data Factory (ADF) ETL Pipeline**

### **Use Case**: Enterprise-scale batch ETL with scheduling, monitoring, and integration.

### **Description**:

Azure Data Factory allows you to build **low-code ETL pipelines** to extract data from SQL Server and load it into **Azure Data Lake Storage (ADLS)** or **Blob Storage**, from where Databricks can ingest the data.

### **Typical Flow**:

1. **ADF Pipeline**: Copy data from SQL Server to ADLS Gen2 in Parquet/CSV.
    
2. **Databricks**: Read files via `read.format("parquet")` or use Autoloader.
    
3. (Optional) Schedule downstream transformations in Databricks notebooks or workflows.
    

### **Key Features**:

- Visual ETL designer
    
- Built-in connectors for SQL Server, ADLS, Databricks
    
- Supports incremental loads using watermark columns
    

---

## 4. **Lakehouse Federation via Foreign Catalog (Virtual Query Only)**

### **Use Case**: Query external SQL Server tables in-place without ingesting data into Databricks.

### **Description**:

Databricks **Lakehouse Federation** allows to register an external SQL Server as a **Foreign Catalog**. It enables **cross-catalog access** for querying but does **not ingest or copy data** into Databricks.

### **Steps**:

```sql
CREATE CONNECTION sqlserver_conn
TYPE sqlserver
OPTIONS (
  host 'sqlserver.example.com',
  port '1433',
  user 'username',
  password 'password',
  database 'mydb'
);

CREATE FOREIGN CATALOG sqlserver_catalog
USING CONNECTION sqlserver_conn;
```

Then query:

```sql
SELECT * FROM sqlserver_catalog.dbo.orders WHERE order_date > '2023-01-01';
```

### **Key Features**:

- No data movement
    
- No storage cost in Databricks
    
- Read-only access
    
- Supports multiple external data sources
    

---

## Summary Table

|Method|Type|Ingestion into Delta|Use Case|Write Support|
|---|---|---|---|---|
|Spark JDBC Connector|Direct|Yes|On-demand/batch loading|Yes|
|Autoloader + Blob Storage|Staged|Yes|Scalable, incremental, schema-evolving loads|Yes|
|Azure Data Factory Pipeline|Orchestrated|Yes|Scheduled ETL, enterprise integration|Yes|
|Foreign Catalog (Federation)|Virtualized|No|Query-only access to external DBs|No|



### 🔍 What is **Database Virtualization** in Databricks?

**Database virtualization** in Databricks refers to the ability to **access and query external databases or data sources in-place** — **without physically moving or copying** the data into Databricks.

In Databricks, this is primarily implemented through  **Lakehouse Federation with Foreign Catalogs**
    

---

## 📌 Example

```sql
-- Create connection to external database
CREATE CONNECTION sqlserver_conn
TYPE sqlserver
OPTIONS (
  host 'sqlserver.example.com',
  port '1433',
  user 'my_user',
  password 'my_password',
  database 'salesdb'
);

-- Create a Foreign Catalog
CREATE FOREIGN CATALOG sqlserver_catalog
USING CONNECTION sqlserver_conn;

-- Query an external table (no data copied)
SELECT * FROM sqlserver_catalog.dbo.orders;
```

---
 ACID is **not guaranteed by Databricks virtualization** because **Databricks does not control or manage the actual data** when using database virtualization.

|Feature|Virtualized (Foreign Catalog)|Managed Delta Table (in Databricks)|
|---|---|---|
|ACID transactions|Governed by external DB|Fully supported via Delta Lake|
|Schema enforcement|Handled by external DB|Enforced by Delta Lake|
|Concurrency control|External DB’s responsibility|Delta transaction log + locks|
|Data versioning|Not available|Supported with Delta Lake|
|Time travel / rollback|Not available|Supported with Delta Lake|
