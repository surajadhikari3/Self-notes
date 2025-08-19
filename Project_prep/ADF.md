

Perfect—here’s a clean T‑SQL script set for **Azure SQL Database** to enable CDC on a simple **Users** table `(Id, Name, Email)` and verify it’s working. Run these in **Azure Data Studio** against your target database (not `master`) as a user with `db_owner`.

---

### 1) Create the table

```sql
CREATE SCHEMA IF NOT EXISTS dbo;

CREATE TABLE dbo.Users
(
  Id    INT            NOT NULL PRIMARY KEY,
  Name  NVARCHAR(100)  NOT NULL,
  Email NVARCHAR(255)  NOT NULL UNIQUE
);
```

### 2) Enable CDC at the **database** level (once per DB)

```sql
EXEC sys.sp_cdc_enable_db;

-- Verify
SELECT name, is_cdc_enabled
FROM sys.databases
WHERE name = DB_NAME();    -- expect is_cdc_enabled = 1
```

### 3) Enable CDC on the **Users** table

> Include only the columns you want captured.

```sql
EXEC sys.sp_cdc_enable_table
  @source_schema        = N'dbo',
  @source_name          = N'Users',
  @role_name            = N'cdc_admin',              -- optional reader role
  @supports_net_changes = 1,                         -- enables net-changes function
  @captured_column_list = N'Id,Name,Email';
```

**Verify table tracking + capture instance**

```sql
SELECT name, is_tracked_by_cdc
FROM sys.tables
WHERE name = 'Users';

EXEC sys.sp_cdc_help_change_data_capture
  @source_schema = N'dbo', @source_name = N'Users';  -- should show cdc.dbo_Users_CT
```

### 4) Generate some test changes

```sql
-- Inserts
INSERT dbo.Users (Id, Name, Email) VALUES
(1, N'Alice', N'alice@example.com'),
(2, N'Bob',   N'bob@example.com');

-- Update
UPDATE dbo.Users
   SET Name = N'Bob Jr.'
 WHERE Id = 2;

-- Delete
DELETE dbo.Users WHERE Id = 1;
```

### 5) Read the CDC changes

Get LSN bounds:

```sql
DECLARE @from_lsn BINARY(10) = sys.fn_cdc_get_min_lsn('dbo_Users');
DECLARE @to_lsn   BINARY(10) = sys.fn_cdc_get_max_lsn();
```

**All changes (every operation row):**

```sql
SELECT *
FROM cdc.fn_cdc_get_all_changes_dbo_Users(@from_lsn, @to_lsn, N'all')
ORDER BY __$start_lsn, __$seqval;
```

**Net changes (one row per PK over interval):**

```sql
SELECT *
FROM cdc.fn_cdc_get_net_changes_dbo_Users(@from_lsn, @to_lsn, N'all')
ORDER BY __$start_lsn;
```

> Key columns in the results:
> 
> - `__$operation` → 1=delete, 2=insert, 3=update (before), 4=update (after)
>     
> - `__$start_lsn`, `__$seqval` → ordering
>     
> - Your captured columns: `Id, Name, Email`
>     

### 6) (Optional) Retention & disable

**Change retention (e.g., keep 3 days of CDC rows):**

```sql
EXEC sys.sp_cdc_change_job
  @job_type = N'cleanup',
  @retention = 4320;  -- minutes = 3 days
```

**Disable (if needed):**

```sql
-- Disable table
EXEC sys.sp_cdc_disable_table
  @source_schema = N'dbo',
  @source_name = N'Users',
  @capture_instance = N'dbo_Users';

-- Disable DB
EXEC sys.sp_cdc_disable_db;
```

---

### Hooking to ADF CDC + Databricks (quick pointers)

- In **ADF CDC**, select `dbo.Users` as **Source**, map **key=Id**, choose **Delta on ADLS** as **Target**, enable **deletes**, and set preferred latency.
    
- In **Databricks**, read the Delta table at your ADLS **abfss://** path; enable **Delta CDF** on that target if you want before/after style events downstream.
    

If you want, tell me your actual schema/table names and I’ll tweak the script exactly to match (and add the ADF mapping screenshot‑style fields).



That error is just SQL syntax—your alias **`current_user`** clashes with the keyword `CURRENT_USER`.

Use this instead:

```sql
-- Who am I and where am I?
SELECT 
  DB_NAME()      AS current_db,
  USER_NAME()    AS db_user,        -- database user
  SUSER_SNAME()  AS server_login;   -- login/identity
```

Next, check if you’re `db_owner`:

```sql
SELECT r.name AS db_role
FROM sys.database_role_members drm
JOIN sys.database_principals r ON r.principal_id = drm.role_principal_id
JOIN sys.database_principals u ON u.principal_id = drm.member_principal_id
WHERE u.name = USER_NAME();
```

If `db_owner` is **missing**, grant it (one of these):

### If you’re Azure AD (and you have AAD admin rights on the server)

```sql
-- run in THIS database
CREATE USER [your.upn@yourtenant.com] FROM EXTERNAL PROVIDER;
ALTER ROLE db_owner ADD MEMBER [your.upn@yourtenant.com];
```

### If you’re using a SQL user that already exists in this DB

```sql
ALTER ROLE db_owner ADD MEMBER [your_sql_user];
```

Now enable CDC:

```sql
EXEC sys.sp_cdc_enable_db;
GO

-- Enable on Users table
EXEC sys.sp_cdc_enable_table
  @source_schema        = N'dbo',
  @source_name          = N'Users',
  @role_name            = N'cdc_admin',
  @supports_net_changes = 1,
  @captured_column_list = N'Id,Name,Email';
GO
```

Verify:

```sql
SELECT name, is_cdc_enabled FROM sys.databases WHERE name = DB_NAME();
SELECT name, is_tracked_by_cdc FROM sys.tables WHERE name='Users';
```

If you want, paste me the outputs of the first two queries (`current_db/db_user/login` and the role list), and I’ll give you the exact `ALTER ROLE` statement with your principal names filled in.