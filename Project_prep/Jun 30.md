

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

| Method                  | Purpose                             |
| ----------------------- | ----------------------------------- |
| Unity Catalog           | Central metadata and governance     |
| Data Explorer           | Visual navigation and search        |
| Global Search           | Fast lookup across workspace        |
| Table & Column Metadata | Understand schema, tags, usage      |
| Data Lineage            | Understand origins and dependencies |
| APIs & SQL Commands     | Automation and scripting            |
|                         |                                     |

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


Visualiztion 


![](../Pasted%20image%2020250630203128.png)

![](../Pasted%20image%2020250630205241.png)

![](../Pasted%20image%2020250630205802.png)

![](../Pasted%20image%2020250630221056.png)


![](../Pasted%20image%2020250702064749.png)

![](../Pasted%20image%2020250702064824.png)


![](../Pasted%20image%2020250702064845.png)

![](../Pasted%20image%2020250702081623.png)


![](../Pasted%20image%2020250702105714.png)



![](../Pasted%20image%2020250702125723.png)


![](../Pasted%20image%2020250702131227.png)



Producer and consumer flow.

1. Producing to Databricks
	1.1 If the file is in cloud storage it can directly be ingested to the databricks using the Autoloader
   1.2  If the file source is in on-prem then it can be batch loaded to the Cloud storage, and using the Autoloader it can be produced to to Databricks to  the Bronze table
   1.3 If the cloud storage is not to be used then CDC has to be manually implemented using the CDC tools like Debezium, Goldengate and publish the event to the kafka and kafka will produce to the Databricks.
2. Once the data is in Databricks it can be produced as delta managed table and can be written into the Bronze table.
   Bronze table act as the source for different consumer and they can process the data as per their requirement and can generate the analytics on it.
3. Role management for differnet users and groups via unity catalog
4. Data cleaning and transformation from bronzeto silver layer
5. Aggregation and some transformation based on the business need and the generation of the Gold layer
6. Generation of visualization, dashboards using the databricks platform itself
7. Extending the visulization , dashboards to the tools like Tabalaue, Power BI, Looker for more enhancement
8. Sharing of the generated Visualization with the analyst or other stake holders. Databricks provide the publishing and role based sharing with other users.


Here's your **enhanced Producer-Consumer flow with Databricks**, updated with missing components, proper terminology, best practices, and **inline code snippets** for each stage based on your image and outline:

---

## ✅ **Enhanced Producer to Consumer Flow (with Code Snippets)**

---

### 🔹 **Step 1: Producing to Databricks**

#### 1.1 If file is in **Cloud Storage** → Use **AutoLoader**:

Autoloader automatically tracks new files from S3, ADLS, or GCS and writes to Bronze.

```python
bronze_df = (
    spark.readStream.format("cloudFiles")
    .option("cloudFiles.format", "json")  # or csv/parquet
    .option("cloudFiles.schemaLocation", "/mnt/schema/bronze/")
    .load("/mnt/raw/events/")
)

bronze_df.writeStream.format("delta") \
    .option("checkpointLocation", "/mnt/checkpoints/bronze/") \
    .start("/mnt/bronze/events")
```

---

#### 1.2 If file is from **On-Prem**:

- Batch upload it to **Cloud Storage** first.
    
- Then use **AutoLoader** as above.
    

📌 **Tip**: Use Azure Data Factory, AWS DataSync, or gsutil to transfer on-prem files to the cloud.

---

#### 1.3 If cloud storage is **not used**, ingest via **CDC tools + Kafka**:

Use tools like **Debezium / GoldenGate** to capture changes and publish to **Kafka**.

```python
kafka_df = (
    spark.readStream.format("kafka")
    .option("kafka.bootstrap.servers", "broker:9092")
    .option("subscribe", "cdc.orders")
    .load()
)

value_df = kafka_df.selectExpr("CAST(value AS STRING)")
bronze_df = spark.read.json(value_df.rdd.map(lambda x: x["value"]))

bronze_df.writeStream.format("delta") \
    .option("checkpointLocation", "/mnt/checkpoints/bronze_kafka/") \
    .start("/mnt/bronze/cdc_orders")
```

---

### 🔹 **Step 2: Bronze Table in Databricks**

- All raw data lands here.
    
- Stored in **Delta Lake** format.
    
- Immutable, append-only.
    

🧠 Acts as the single source of truth for downstream processing.

```python
# Example Bronze Query
spark.read.format("delta").load("/mnt/bronze/events").display()
```

---

### 🔹 **Step 3: Role Management with Unity Catalog**

- Unity Catalog handles:
    
    - Table-level, column-level, row-level access control
        
    - Governance (lineage, audit, tags)
        
- Integrated with identity providers like Azure AD
    

```sql
GRANT SELECT ON TABLE bronze.events TO `analyst_group`;
```

---

### 🔹 **Step 4: Silver Table (Cleaned and Enriched)**

Transforms Bronze data:

- Cleans nulls
    
- Fixes formats
    
- Deduplicates
    
- Adds reference data
    

```python
bronze_df = spark.read.format("delta").load("/mnt/bronze/events")

silver_df = (
    bronze_df
    .filter("event_type IS NOT NULL")
    .withColumn("event_time", to_timestamp("event_time"))
    .dropDuplicates(["event_id"])
)

silver_df.write.format("delta").mode("overwrite").save("/mnt/silver/cleaned_events")
```

---

### 🔹 **Step 5: Gold Table (Aggregated Business KPIs)**

Performs business-level aggregations:

- KPIs, metrics, reporting summaries
    

```python
silver_df = spark.read.format("delta").load("/mnt/silver/cleaned_events")

gold_df = (
    silver_df.groupBy("user_id")
    .agg(
        count("*").alias("event_count"),
        max("event_time").alias("last_event_time")
    )
)

gold_df.write.format("delta").mode("overwrite").save("/mnt/gold/user_summary")
```

---

### 🔹 **Step 6: Dashboards in Databricks**

- Use **SQL Editor → Visualizations** to build dashboards from Gold Tables.
    
- Supports charts, time series, KPIs, pivot tables.
    

📌 Dashboards can be shared with users/groups based on Unity Catalog permissions.

---

### 🔹 **Step 7: External Visualization (Power BI, Tableau, Looker, Qlik)**

- Use **JDBC/ODBC connectors** or **SQL Warehouses** to connect BI tools.
    

Example: Tableau connection string to Databricks SQL endpoint.

```bash
jdbc:spark://<databricks-sql-endpoint>:443/default;transportMode=http;ssl=1;httpPath=<path>
```

---

### 🔹 **Step 8: Stakeholder Sharing**

- Dashboards can be **published directly** from Databricks.
    
- Shared via role-based access (analyst, execs).
    
- Also accessible via BI tools (if integrated).
    

---

### 🧭 Summary Table

|Step|Layer|Purpose|Technology|Code Snippet Included|
|---|---|---|---|---|
|1|Ingest|Raw data into Databricks|Autoloader / Kafka|✅|
|2|Bronze|Raw Delta Table|Delta Lake|✅|
|3|Access|Role-based control|Unity Catalog|✅|
|4|Silver|Cleaned Data|PySpark/SQL|✅|
|5|Gold|Aggregated Data|PySpark/SQL|✅|
|6|Visualization|Internal Dashboards|Databricks Viz|⚡|
|7|External Viz|BI Integration|Power BI, Tableau|✅|
|8|Sharing|Stakeholder Access|SQL/ACLs|✅|


---

## ✅ Step 8: Generate and Share Analytics (SQL Queries, Visualizations, Dashboards) via API

Once data is available in the Gold layer (transformed Delta tables), you can programmatically create and share analytics using the following API-based flow:

### 🧭 8.1. Create a SQL Query

Use the Queries API to define your SQL logic:

```bash
curl -X POST https://<DATABRICKS_INSTANCE>/api/2.0/sql/queries \
  -H "Authorization: Bearer <TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Revenue by Region",
    "query": "SELECT region, SUM(sales) as revenue FROM gold.sales GROUP BY region",
    "data_source_id": "<SQL_WAREHOUSE_ID>"
  }'
```

🔁 Store the query_id from the response.

---

### 📊 8.2. Create a Visualization (e.g., Bar Chart)

Attach a chart to that query using the Visualizations API:

```bash
curl -X POST https://<DATABRICKS_INSTANCE>/api/2.0/sql/visualizations \
  -H "Authorization: Bearer <TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "barchart",
    "name": "Revenue Chart",
    "query_id": "<query_id>",
    "options": {
        "xColumn": "region",
        "yColumn": "revenue",
        "aggregation": "SUM"
    }
  }'
```

🔁 Store the visualization_id from the response.

---

### 📋 8.3. Create or Reuse a Dashboard

Use the Dashboards API to create a container for the visual:

```bash
curl -X POST https://<DATABRICKS_INSTANCE>/api/2.0/sql/dashboards \
  -H "Authorization: Bearer <TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{ "name": "Sales Overview Dashboard" }'
```

🔁 Store dashboard_id from the response.

---

### 🧱 8.4. Add the Visualization to the Dashboard (as a Widget)

Databricks supports adding visuals via internal widget APIs (currently not well-documented externally), but can be automated using the UI and shared via permission API (next step).

---

### 🔐 8.5. Share Dashboard or Query with Teams

Use the Permissions API to grant access to consumers (analysts, BI teams, business users):

```bash
curl -X PATCH https://<DATABRICKS_INSTANCE>/api/2.0/permissions/sql/dashboard/<dashboard_id> \
  -H "Authorization: Bearer <TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{
    "access_control_list": [
      {
        "group_name": "executive_team",
        "permission_level": "CAN_VIEW"
      }
    ]
  }'
```

---

✅ Result: The data consumer group now sees the full dashboard with charts in their Databricks SQL workspace, based on governed Gold-layer data.

---
