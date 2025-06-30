

Data Discovery by Cataloging



In **Databricks**, data discovery is the process of finding, understanding, and managing datasets available within your workspace or shared environment. Databricks supports **data discovery** through **Unity Catalog**, **Data Explorer**, **search features**, **lineage tracking**, and **governance tools**.

---

### 🔍 1. **How Data Discovery Happens in Databricks**

|Component/Tool|Role in Data Discovery|
|---|---|
|**Unity Catalog**|Centralized governance and metadata layer. Supports cataloging, tagging, access control, and lineage.|
|**Data Explorer UI**|Browse catalogs → schemas → tables → views. Interactive way to explore datasets.|
|**Search Bar**|Global search lets you find tables, notebooks, dashboards by name or tag.|
|**Lineage View**|Shows how a dataset was created and which downstream assets consume it.|
|**Column-Level Metadata**|Each column can have data types, comments, and tags to help identify its purpose.|
|**Tags & Annotations**|Helps in classifying and categorizing data for easy filtering and discovery.|
|**Notebook Metadata**|Users can document ETL pipelines, transformations, and usage notes alongside the data.|

---

### 🔐 2. **Role of Unity Catalog in Discovery**

Unity Catalog plays a central role in data discovery by:

- **Organizing data assets** into:
    
    - Catalogs
        
    - Schemas
        
    - Tables/Views
        
- **Governance features** like:
    
    - Role-based access control (RBAC)
        
    - Table/column-level permissions
        
    - Attribute-based tagging (e.g., PII, Finance)
        
- **Searchability** via UI & API
    
- **Data lineage view** (including notebooks, DLT pipelines, jobs)
    

---

### 📈 3. **Workflow Example: Discovering Data**

Imagine a user wants to find sales data:

1. **Open Data Explorer** in Databricks workspace.
    
2. Navigate:
    
    - `SalesCatalog` → `MonthlySchema` → Tables
        
3. Search or filter using tags like `"finance"`, `"2024"`.
    
4. Click on table → see description, schema, data preview, lineage.
    
5. Explore how this table was generated or used via **lineage tab**.
    
6. Optionally, query it in SQL editor or attach to notebook.
    

---

### 🔄 4. **Discoverability in Streaming/Data Pipelines**

When using **Delta Live Tables (DLT)** or **Auto Loader**:

- The assets are automatically **registered in Unity Catalog** (if configured).
    
- They appear in lineage views and **metadata is auto-populated**.
    
- Can be tagged or documented by developers.
    

---

### 📘 5. **Metadata API / Programmatic Discovery**

You can programmatically discover data using:

```sql
SHOW CATALOGS;
SHOW SCHEMAS IN my_catalog;
SHOW TABLES IN my_catalog.my_schema;
DESCRIBE TABLE my_catalog.my_schema.sales_data;
```

Or through **Unity Catalog REST APIs** for automation, lineage, and metadata extraction.

---

### ✅ Summary Table

|Method|Purpose|
|---|---|
|Unity Catalog|Central metadata and governance|
|Data Explorer|Visual navigation and search|
|Global Search|Fast lookup across workspace|
|Table & Column Metadata|Understand schema, tags, usage|
|Data Lineage|Understand origins and dependencies|
|APIs & SQL Commands|Automation and scripting|

![](../Pasted%20image%2020250630163226.png)



![](../Pasted%20image%2020250630163507.png)


---

 how to get data from Oracle / Mysql / Mongo into DataBricks catalog -> schema -> table structure

- do we do CDC etc.


Reading data from an **on-premise database** (like Oracle, PostgreSQL, MySQL, SQL Server) into **Databricks** can be done using **JDBC** or **CDC-based ingestion tools**. Here's a comprehensive guide covering **both approaches** with **hands-on steps**:

---

## ✅ Option 1: ** Batch Injection Using JDBC (Batch or Incremental Pull)**

### 🔧 Prerequisites:

- Ensure **network connectivity** from Databricks to your on-prem DB (via VPN, private link, or tunneling).
    
- Ensure the **JDBC driver** is available (Databricks provides many built-in).
    

---

### 🧪 Example: Read from PostgreSQL (Replace with Oracle URL for Oracle)

```python
jdbcDF = spark.read \
    .format("jdbc") \
    .option("url", "jdbc:postgresql://<on-prem-host>:5432/<db>") \
    .option("driver", "org.postgresql.Driver") \
    .option("dbtable", "public.employees") \
    .option("user", "my_user") \
    .option("password", "my_password") \
    .load()

display(jdbcDF)
```

> For **Oracle**, use:  
> `jdbc:oracle:thin:@<host>:1521/<service_name>`  
> Driver: `"oracle.jdbc.driver.OracleDriver"`

---

### 🕒 To Load Only New Records (Manual CDC):

```python
# Suppose 'last_updated' is a timestamp column
incremental_df = spark.read \
    .format("jdbc") \
    .option("url", "jdbc:postgresql://<host>:5432/<db>") \
    .option("dbtable", f"(SELECT * FROM public.employees WHERE last_updated > '{last_run_time}') AS temp") \
    .option("user", "user") \
    .option("password", "pass") \
    .load()
```

You’ll need to **store `last_run_time`** in a config table or Delta table.

---

## ✅ Option 2: **Using Debezium + Kafka (Recommended for Streaming CDC)**

If you want **real-time CDC streaming**, integrate via **Debezium** running close to your on-prem DB and push changes to **Kafka**, then stream into Databricks.

---

### ⚙️ Architecture:

```text
On-Prem Oracle/Postgres
      ↓
Debezium Connector (captures INSERT/UPDATE/DELETE)
      ↓
Kafka Topic
      ↓
Databricks Structured Streaming
```

---

### 🧪 Databricks Code to Read Kafka CDC Stream:

```python
df = (spark.readStream
      .format("kafka")
      .option("kafka.bootstrap.servers", "kafka-broker:9092")
      .option("subscribe", "employee_cdc_topic")
      .load())

json_df = df.selectExpr("CAST(value AS STRING)")
```

Debezium CDC messages are JSON—use `from_json()` to extract and transform.

---

## ✅ Option 3: **ETL Tools (If VPN setup is complex)**

Use third-party tools to move data:

- **Fivetran, Talend, Informatica**: Extract from on-prem → Cloud storage.
    
- Then use **Autoloader** in Databricks to ingest incrementally from cloud.
    

---

## 🔒 Security Note:

- Always use **SSL/JDBC encryption** when connecting over the internet.
    
- Prefer **Azure Private Link or VPN tunnels** for connectivity.
    

---

## 📝 Summary Table:

|Approach|Use Case|Pros|Cons|
|---|---|---|---|
|JDBC (batch)|Simple one-time or scheduled pulls|Quick to implement|Not scalable for large data|
|JDBC (incremental)|Pull only changed records via timestamp|Less load than full scans|Manual state mgmt required|
|Kafka + Debezium|Real-time, reliable change stream|Highly scalable, reliable CDC|Infra setup needed|
|Fivetran/Informatica|No-code/Low-code connectors|Easy to set up|May be costly|

---

For the On prem Autoloader does not support so have to manually implement the CDC (Kafka + Debezium) or Use the Out-box pattern.

No Autoloader only works with cloud storage object for CDC . Need to read it as a cloudFiles Format..

```
df = (spark.readStream
      .format("cloudFiles")
      .option("cloudFiles.format", "json")  # or csv/parquet
      .load("s3://bucket/path/"))
```



There is the data visualization, Dashboard in the Databricks. Is it Better than the Tableaue, Power BI as it also provide the connector ? 



| Scenario                                                      | Use This                  |
| ------------------------------------------------------------- | ------------------------- |
| Build quick dashboards directly on top of Delta/Unity Catalog | **Databricks Dashboards** |
| Share dashboards with non-technical users company-wide        | **Power BI or Tableau**   |
| Need beautiful, highly customized dashboards                  | **Tableau**               |
| You're already using Microsoft 365 ecosystem                  | **Power BI**              |

## **Databricks Dashboard vs Tableau vs Power BI**

|Feature|**Databricks Dashboard**|**Tableau**|**Power BI**|
|---|---|---|---|
|**Native to Databricks**|✅ Yes|❌ (external tool, requires connector)|❌ (external tool, requires connector)|
|**Real-Time with Delta**|✅ Excellent (uses Delta Live/streaming)|⚠️ Possible, with some latency|⚠️ Limited, based on refresh settings|
|**Advanced Visuals**|❌ Basic charts|✅ Rich and interactive|✅ Rich and customizable|
|**AI-Powered Insights**|⚠️ Limited|✅ With extensions|✅ Built-in with Q&A (NLP support)|
|**Data Governance Integration**|✅ Unity Catalog + ACLs|⚠️ External to Databricks|⚠️ External to Databricks|
|**Cost**|✅ Included in Databricks platform|💰 Separate licensing|💰 Separate (Power BI Pro/Premium)|
|**Ease for BI/Analysts**|⚠️ Good for engineers, not analysts|✅ Analyst-friendly|✅ Very analyst-friendly|
|**Embedding/Sharing**|⚠️ Basic dashboards|✅ Excellent (embedding + publishing)|✅ Excellent (Power BI service, Teams)|
|**Offline Mode**|❌ Not supported|✅ Supported|✅ Supported|


|Criteria|Recommendation|
|---|---|
|You’re a data engineer|✅ Databricks Dashboards|
|You’re a business analyst|✅ Power BI or Tableau|
|Want deep integration with Azure/Microsoft|✅ Power BI|
|Want beautiful, interactive dashboards|✅ Tableau|
|Need real-time streaming insights from Delta|✅ Databricks Dashboards|
|Cost and speed is a concern|✅ Stick with Databricks dashboard (no extra licensing)|
