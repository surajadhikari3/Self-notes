![](../Pasted%20image%2020250619101030.png)


# How is Unity Catalog doing cataloging of data in databricks?

Unity Catalog in **Databricks** is a unified governance solution that provides **fine-grained access control, centralized metadata management**, and **data lineage tracking** across **workspaces** and **cloud platforms** (AWS, Azure, GCP). It acts as a **central catalog service** for managing all data assets—tables, views, volumes, functions, files, and ML models.

---

### 🔍 **How Unity Catalog Does Cataloging of Data**

Unity Catalog introduces a **three-level namespace** to organize and manage data:

```
catalog.schema.table
```

#### ✅ **Cataloging Workflow Overview**

1. **Metastore**
    
    - Unity Catalog is backed by a centralized **metastore**, which holds metadata about all the data assets.
        
    - This metastore is configured per region and can serve multiple Databricks workspaces.
        
    - Admins create **catalogs** in the metastore.
        
2. **Catalogs**
    
    - A catalog is the **top-level container** for data assets (like a database group).
        
    - Each catalog contains **schemas** (like databases).
        
3. **Schemas (Databases)**
    
    - A schema contains **tables**, **views**, **functions**, and **volumes**.
        
    - It also defines ownership and access privileges at the schema level.
        
4. **Tables and Views**
    
    - Tables can reference data stored in **Delta Lake**, external sources (Parquet, CSV, JDBC), or volumes (object stores).
        
    - When a table is created, **metadata like schema, location, owner, and permissions** is stored in Unity Catalog.
        

---

### 📊 **How Unity Catalog Tracks and Catalogs Metadata**

|Feature|Role in Cataloging|
|---|---|
|**Centralized Metastore**|Manages all metadata across workspaces and cloud providers.|
|**Data Object Metadata**|Tracks columns, data types, table location, creation time, creator, etc.|
|**Table Lineage**|Captures relationships between input and output tables to provide full data lineage.|
|**Volume Metadata**|Catalogs files and folders under managed Volumes (cloud object storage).|
|**External Locations**|Enables referencing external cloud storage while keeping it governed.|
|**Privileges**|Tracks ACLs using ANSI GRANT statements on catalogs, schemas, tables, and views.|

---


--> In short it is done via Metastore and catlog which is the logical grouping of schema (It added one more layer of catlog and make it 3 layer)
 metastore is configured per region and can serve multiple Databricks workspaces.



so lets say that the metastore is region based and the two different table is running in the different metastore in that case how the catloging is maintained by databricks?

---

## 📍 Key Principle: Unity Catalog Metastore Is **Region-Bound**

- Each **Unity Catalog Metastore** is tied to **a single cloud region** (e.g., `east-us`, `west-europe`).
    
- A **workspace can only be attached to one metastore** at a time.
    
- A **metastore governs all the catalogs/schemas/tables** within its assigned region.
    

---

## 🔁 Scenario: Two Tables in Different Regions

### Let's say:

- **Table A** is in a catalog under **Metastore-1 (Region: `us-east-1`)**
    
- **Table B** is in another catalog under **Metastore-2 (Region: `us-west-2`)**
    

### 🧠 How Unity Catalog handles this:

### ❌ **There is _no_ cross-metastore cataloging** in Unity Catalog.

- **Each metastore is fully isolated**.
    
- You **cannot query or join tables** across different metastores directly.
    
- Each metastore maintains its **own namespace**, access controls, lineage, and metadata.
    

---

## ✅ How to Work With Multi-Metastore Architectures

### 💡 Option 1: Co-locate Data in One Region

- Recommended for most use cases.
    
- If latency or compliance allows, **migrate data into a common metastore** (e.g., replicate table from Region B to Region A).
    

### 💡 Option 2: Use External Query Federation (e.g., Partner Tools)

- Tools like **Starburst**, **Trino**, or **Databricks Partner Connect** can federate queries across regions, but you lose native Unity Catalog support like fine-grained lineage and permissions.
    

### 💡 Option 3: Use Delta Sharing (Advanced)

- Use **Delta Sharing** to **expose tables from one metastore to another** securely.
    
- Consumer workspaces can access shared tables **as if they were native**, but **read-only**.
    

---

## 🧱 Summary: How Cataloging Is Maintained Per Metastore

|Feature|Metastore-Scoped|
|---|---|
|Catalogs|✅ Yes|
|Schemas & Tables|✅ Yes|
|Access Policies (RBAC)|✅ Yes|
|Lineage Tracking|✅ Yes|
|Audit Logs|✅ Yes|
|Cross-Region Access|❌ Not supported directly|

---

### 🌐 Visual Representation

```
Metastore A (us-east-1)
│
├── Catalog: sales_data
│   └── Schema: east_coast_orders
│       └── Table: orders_2024

Metastore B (us-west-2)
│
├── Catalog: inventory_data
│   └── Schema: west_warehouse
│       └── Table: stock_2024

⚠️ Cannot join or catalog across Metastore A and B directly.
```

---

## 🔐 Best Practices for Multi-Region Strategy

- Standardize on **one metastore per geo boundary or business unit**.
    
- Use **naming conventions** to reflect data ownership (e.g., `us_east_catalog`, `eu_analytics_catalog`).
    
- Use **Delta Sharing** or **ETL jobs** for cross-region replication.
    
- Implement **data governance** policies regionally via separate Unity Catalog configurations.
    

---


![](../Pasted%20image%2020250625154439.png)





Questions to be asked: 

Does the access managment needs to handle at the reporting or catalog level?

In the settings -> Advanced Settings -> Admin can give the access control, Personal Access Token (PAT) for authorization 

# How the Databricks does the access management.?



![](../Pasted%20image%2020250625161122.png)


Can also provide the Table based access from the security ..


Access control in **Databricks** is governed by **Unity Catalog**, which provides **fine-grained governance** for data and AI assets, enabling secure collaboration across teams. Here’s a structured view of **how access control works in Databricks**, especially across components like catalogs, schemas, tables, and beyond:


![](../Pasted%20image%2020250625162500.png)

---

1. Access Management from the Databrics


### ✅ 1. **Access Control Flow in Databricks (Unity Catalog)**

```text
User or Service Principal
        ↓
Authentication (SSO, Azure AD, SCIM)
        ↓
Authorization (Unity Catalog ACLs + Workspace Access Control)
        ↓
Access to:
  - Catalogs
  - Schemas (Databases)
  - Tables, Views, Functions
  - Notebooks, Clusters, Jobs
```

---

### ✅ 2. **Authentication**

Databricks supports authentication via:

- Azure Active Directory (Azure AD)
    
- SCIM(System for Cross-domain Identity Management) (for user/group sync)
    
- Personal Access Tokens (PATs)
    
- Service Principals (for non-interactive apps)

![](../Pasted%20image%2020250625161122.png)

---

### ✅ 3. **Authorization (Access Control Levels)**

Unity Catalog introduces a **three-level hierarchical model**:

| Level          | Description                            | Example Permissions                     |
| -------------- | -------------------------------------- | --------------------------------------- |
| **Catalog**    | Top-level namespace (like a DB server) | `USE CATALOG`, `CREATE SCHEMA`, `GRANT` |
| **Schema**     | Like a database within a catalog       | `CREATE TABLE`, `SELECT`, `MODIFY`      |
| **Table/View** | Actual data objects                    | `SELECT`, `INSERT`, `UPDATE`, `DELETE`  |

 Access Control  From Ui
![](../Pasted%20image%2020250625161621.png)


Access Control line From CommandLine 

```sql
-- Granting access to a table
GRANT SELECT ON TABLE main.my_catalog.sales.orders TO `data_analyst_group`;

-- Granting usage of schema
GRANT USE SCHEMA ON SCHEMA main.my_catalog.sales TO `data_analyst_group`;

-- Granting usage of catalog
GRANT USE CATALOG ON CATALOG main.my_catalog TO `data_analyst_group`;
```
#### 🟩 Example Flow:

- A user must have `USE CATALOG` on the catalog → `USE SCHEMA` on the schema → `SELECT` on the table to run a query.
    

---

### ✅ 4. **Who Grants the Access?**

- **Metastore Admin** or **Data Owner** grants access using GRANT statements or the Databricks UI.
    
- Privileges are **cascading** – you need upstream access (catalog/schema) to access downstream objects (tables/views).
    

---

### ✅ 5. **How Access Flows Across Components**

#### 🔸 Notebook / Job Access:

- If a notebook queries a Unity Catalog table, the **notebook runner's identity** (user or service principal) is used.
    
- Access to the underlying table is evaluated based on the Unity Catalog ACLs.
    

#### 🔸 Clusters:

- Unity Catalog supports **secure clusters** which enforce table-level access.
    
- These clusters use **credential passthrough** or **identity federation** to impersonate users and apply ACLs properly.
    
    

---

### ✅ 6. **Auditing and Governance**

- All access is **logged and auditable**.
    
- Unity Catalog integrates with **Databricks audit logs**, which can be sent to cloud-native monitoring systems or SIEMs.
    

---

### ✅ 7. **Access Control Management Options**

You can manage ACLs via:

|Method|Description|
|---|---|
|Databricks UI|Click-based permission management|
|SQL Commands|`GRANT`, `REVOKE`, `SHOW GRANTS`|
|Terraform / API|Infra as code for governance|

---


 2.   Persona Mangement using  Teraform --> Can define the template providing the role access to respective user
(https://www.databricks.com/discover/pages/access-control#introHear )
---

### 🧠 Summary

|Component|Access Controlled At|Notes|
|---|---|---|
|Catalog|`USE CATALOG`, `CREATE SCHEMA`|Top-level container|
|Schema (Database)|`USE SCHEMA`, `CREATE TABLE`|Middle layer|
|Table/View|`SELECT`, `MODIFY`, `ALL PRIVILEGES`|Data layer|
|Notebooks|Workspace ACLs + Data ACLs|Uses cluster’s user context|
|Clusters|Shared vs Secure|Secure clusters enforce UC rules|
|External Storage|External Location ACLs|Controlled via Unity Catalog objects|

------------------------------------------------------------------------------------

## 🛠️   **How Are GRANTS Provided? (SQL vs UI)**

|Level|Controlled By|Access Method|Notes|
|---|---|---|---|
|**Catalog**|Unity Catalog ACL|✅ SQL Only|`GRANT USAGE ON CATALOG catalog_name TO user/group`|
|**Schema**|Unity Catalog ACL|✅ SQL Only|`GRANT USAGE ON SCHEMA catalog.schema TO user/group`|
|**Table/View**|Unity Catalog ACL|✅ SQL Only|`GRANT SELECT ON TABLE catalog.schema.table TO user/group`|
|**Row/Column**|Secure Views|✅ SQL Only|Define views using filters like `WHERE region = current_user()`|
|**Notebooks / Dashboards**|Workspace ACL|✅ UI + ✅ SQL|Via UI (share, run access) or API|

⚠️ **Only dashboard/notebook access** can be granted via the **UI**, all **data-level access is SQL-only**.


From ui the access can be managed as shown below for both notebook and dashboard and for the rest the access is granted via sql level................ 

Go to Share and select the access level 

![[Pasted image 20250626132731.png]]





# If the underlying datasource changes, how are the reports affected


---

### 🧩 Scenario: Dashboard on Top of Data in Databricks

You have:

- Data stored as **Delta Tables**
    
- Dashboard created using **Databricks SQL** or connected BI tools
    
- Schema of the table **changes** (column renamed, dropped, added)
    

---

### 🔁 How Databricks Handles Schema Changes

#### ✅ 1. **Delta Lake’s Schema Evolution**

Databricks uses **Delta Lake**, which supports **schema evolution** and **schema enforcement**.

- If you **append data with new columns**, you can enable:
    
    ```sql
    SET spark.databricks.delta.schema.autoMerge.enabled = true;
    ```
    
    This lets Delta **evolve** its schema to include new columns.
    
- If you **drop or rename a column**, that's a **breaking change** for dashboards unless explicitly handled.
    

---

### 🔎 2. **Impact on Dashboards**

|Type of Change|Dashboard Impact|
|---|---|
|**New column added**|Safe. Won’t affect existing visualizations unless explicitly used|
|**Column renamed**|🔴 Breaks charts/queries referencing old name|
|**Column dropped**|🔴 Breaks charts/queries, throws errors|
|**Column data type changed**|⚠️ Might break aggregations or filters based on that column|

---

### 📊 BI Tool Behavior (Power BI / Tableau / Databricks SQL Dashboards)

- **Power BI** and **Tableau** cache metadata — renaming/dropping columns will break reports unless the model is refreshed or updated.
    
- **Databricks SQL Dashboards** will show errors in tiles that reference missing fields.
    

---

### 🛠️ Mitigation Strategies

#### ✅ A. Use **Views as a Semantic Layer**

Create **stable views** over tables to shield dashboards from direct schema changes.

```sql
CREATE OR REPLACE VIEW reporting.sales_summary AS
SELECT
  order_id,
  customer_id,
  quantity,
  total_amount
FROM bronze_layer.sales_raw;
```

> If `sales_raw` changes, you only need to **update the view** — not every dashboard.

---

#### ✅ B. Enable Schema Change Alerts

- You can **monitor schema changes** using Unity Catalog's **table history** (`DESCRIBE HISTORY`) or audit logs.
    
- You can even schedule a notebook to check for schema diffs and trigger alerts.
    

---

#### ✅ C. Track Data Lineage (with Unity Catalog + DLT)

- Databricks tracks **column-level lineage** via **Unity Catalog + Delta Live Tables**.
    
- If a schema changes, you can visualize **what dashboards, notebooks, or queries are impacted**.
    

---

#### ✅ D. Best Practices

- Avoid breaking changes (e.g., dropping/renaming columns) without updating consumers.
    
- Maintain **semantic views** for BI.
    
- Maintain version control over schema (with JSON schema or tags in metadata).
    

---

### 📥 Diagram (Simplified Workflow)

```
         Delta Table
           |
     +-----+------+
     |            |
  Raw Layer   Transformed Layer (Silver/Gold Views)
                    |
              Dashboards (BI, SQL, PowerBI)
```

---



We want to allow the user to add but not the delete and to change the meta-deta? 

### 🛡️ Mitigation Against Deletion or Renaming

Databricks currently doesn’t let you grant `ALTER` privileges **partially** (e.g., only allow ADD but not DROP or RENAME). So, you use this **workaround**:

1. **Do not grant `ALTER` directly**
    
2. **Provide a utility notebook or API endpoint** for controlled schema changes (you validate it allows only `ADD COLUMN`)
    
3. Use audit logs to track changes to schema with:
    
    
    `DESCRIBE HISTORY my_catalog.my_schema.my_table;`


Summary is that :

-> With the Delta Lake it allows the schema evolution --> Can perform Read, write operation
-> It can not explicitly control the delete and read operation . So have to provide utility notebook or API endpoint for controlled schema changes..



Even the dremio does not support this  (So have to handle explicitly on both )


### ❌ What Dremio _Does Not_ Support (as of now)

|Feature You Want|Support in Dremio?|Notes|
|---|---|---|
|Allow **only column addition**|❌ No|No native ACL at column-level granularity for schema evolution|
|Disallow **column deletion only**|❌ No|If a user has `ALTER`, they can modify/drop/add columns freely|
|Column-level privilege enforcement|⚠️ Limited to `SELECT` projections|Not for schema manipulation|




Reporting -> 2025-06-26


Data bricks visualization 

![[Pasted image 20250626113301.png]]


![[Pasted image 20250626115026.png]]



![[Pasted image 20250626120442.png]]


Visualization can also be done directly from the notebook once the table is executed there is the option to add the visualization  

![[Pasted image 20250626122318.png]]


Medallion Architecture

It is a data design pattern that organizes data within a lake-house into layers(zones) to improve quality and accessibility.

It's a common approach to building data pipelines, often referred to as "multi-hop" because data progresses through distinct stages. The three main layers are Bronze (raw data), Silver (validated data), and Gold (enriched data).

![[Pasted image 20250626125120.png]]

![[Pasted image 20250626134329.png]]