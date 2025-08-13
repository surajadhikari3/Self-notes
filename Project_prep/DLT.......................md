


Perfect—DLT can give you **bitemporality out-of-the-box** by combining:

- **Valid time** → SCD Type 2 intervals managed by DLT (`__START_AT`, `__END_AT`) via `AUTO CDC / APPLY CHANGES`.
    
- **System time** → Delta’s time travel / table history (commit timeline).
    

Below is a **drop-in DLT pipeline** that starts from your bronze table with the Debezium `payload` string, parses it, and builds an SCD2 (“bitemporal”) gold table.

---

# Option A — DLT in **SQL** (quick to wire)

> Replace the source table name with your bronze table (I used the one from your screenshot flow).

```sql
-- 0) SOURCE: ingest raw bronze into DLT
CREATE STREAMING LIVE TABLE bronze_raw
AS
SELECT payload
FROM c0a91_centralus_tdvib_tdsbi_ceatalog_bronze.debezium_user_event;

-- 1) PARSE: turn Debezium envelope into row-level CDC (id, name, email, event_ts, op)
CREATE STREAMING LIVE TABLE cdc_rows
AS
WITH env AS (
  SELECT
    get_json_object(payload, '$.payload.op')                       AS op,
    timestamp_millis(CAST(get_json_object(payload, '$.payload.ts_ms') AS BIGINT)) AS event_ts,
    get_json_object(payload, '$.payload.after')  AS after_json,
    get_json_object(payload, '$.payload.before') AS before_json
  FROM STREAM(LIVE.bronze_raw)
  WHERE payload IS NOT NULL AND length(trim(payload)) > 0
),
row_choice AS (
  SELECT
    op, event_ts,
    CASE WHEN op = 'd' THEN before_json ELSE after_json END AS row_json
  FROM env
)
SELECT
  (op = 'd') AS is_delete,
  op,
  event_ts,
  r.*
FROM (
  SELECT
    op, event_ts,
    from_json(
      row_json,
      'struct<id:int,name:string,email:string,created_at:string>'
    ) AS r
  FROM row_choice
) t
WHERE r IS NOT NULL;

-- 2) TARGET: declare the SCD2 (bitemporal) table schema
--    NOTE: __START_AT and __END_AT must match the SEQUENCE BY type (TIMESTAMP here)
CREATE STREAMING LIVE TABLE users_scd2
TBLPROPERTIES (delta.enableChangeDataFeed = true)
(
  id INT,
  name STRING,
  email STRING,
  created_at STRING,
  __START_AT TIMESTAMP,
  __END_AT   TIMESTAMP
);

-- 3) BITEMPORAL: apply changes as SCD Type 2
AUTO CDC INTO LIVE.users_scd2
FROM STREAM(LIVE.cdc_rows)
KEYS (id)
SEQUENCE BY event_ts
APPLY AS DELETE WHEN is_delete = true
STORED AS SCD TYPE 2
COLUMNS * EXCEPT (is_delete, op, event_ts);
```

- `STORED AS SCD TYPE 2` makes DLT maintain **valid-time** windows via `__START_AT` / `__END_AT`. ([Databricks Documentation](https://docs.databricks.com/aws/en/dlt-ref/dlt-sql-ref-apply-changes-into?utm_source=chatgpt.com "AUTO CDC INTO (Lakeflow Declarative Pipelines)"), [Microsoft Learn](https://learn.microsoft.com/en-us/azure/databricks/dlt-ref/dlt-sql-ref-apply-changes-into?utm_source=chatgpt.com "AUTO CDC INTO (Lakeflow Declarative Pipelines)"))
    
- We predeclared `__START_AT` & `__END_AT` in the target schema as **TIMESTAMP** (same type as `event_ts`) per docs. ([Databricks Documentation](https://docs.databricks.com/aws/en/dlt-ref/dlt-sql-ref-apply-changes-into?utm_source=chatgpt.com "AUTO CDC INTO (Lakeflow Declarative Pipelines)"), [Microsoft Learn](https://learn.microsoft.com/en-us/azure/databricks/dlt-ref/dlt-python-ref-apply-changes?utm_source=chatgpt.com "create_auto_cdc_flow - Azure Databricks"))
    
- `APPLY AS DELETE WHEN` treats Debezium deletes correctly. ([Databricks Documentation](https://docs.databricks.com/aws/en/dlt-ref/dlt-sql-ref-apply-changes-into?utm_source=chatgpt.com "AUTO CDC INTO (Lakeflow Declarative Pipelines)"))
    

Once the pipeline runs (set your **pipeline target** to a UC catalog/schema), you can query:

```sql
-- Current state (valid & system “now”)
SELECT * FROM <catalog>.<schema>.users_scd2 WHERE __END_AT IS NULL;

-- Valid-time as-of query
SELECT * FROM <catalog>.<schema>.users_scd2
WHERE __START_AT <= TIMESTAMP('2025-08-10 00:00:00')
  AND ( __END_AT IS NULL OR __END_AT >= TIMESTAMP('2025-08-10 00:00:00') );

-- System-time (time travel) example
SELECT * FROM <catalog>.<schema>.users_scd2 VERSION AS OF 15;
```

Time travel & history give you the **system-time** axis. ([Microsoft Learn](https://learn.microsoft.com/en-us/azure/databricks/delta/history?utm_source=chatgpt.com "Work with Delta Lake table history - Azure Databricks"), [Databricks](https://www.databricks.com/blog/2019/02/04/introducing-delta-time-travel-for-large-scale-data-lakes.html?utm_source=chatgpt.com "Introducing Delta Time Travel for Large Scale Data Lakes"))

---

# Option B — DLT in **Python** (same logic)

```python
import dlt
from pyspark.sql.functions import col, get_json_object, from_json
from pyspark.sql.types import StructType, StructField, IntegerType, StringType

after_schema = StructType([
    StructField("id", IntegerType(), True),
    StructField("name", StringType(), True),
    StructField("email", StringType(), True),
    StructField("created_at", StringType(), True),
])

@dlt.view
def bronze_raw():
    return spark.readStream.table(
        "c0a91_centralus_tdvib_tdsbi_ceatalog_bronze.debezium_user_event"
    ).selectExpr("CAST(payload AS STRING) AS payload")

@dlt.table
def cdc_rows():
    env = dlt.read_stream("bronze_raw").selectExpr(
        "get_json_object(payload, '$.payload.op') AS op",
        "timestamp_millis(CAST(get_json_object(payload, '$.payload.ts_ms') AS BIGINT)) AS event_ts",
        "get_json_object(payload, '$.payload.after')  AS after_json",
        "get_json_object(payload, '$.payload.before') AS before_json"
    )
    rows = env.selectExpr(
        "CASE WHEN op='d' THEN before_json ELSE after_json END AS row_json",
        "op", "event_ts"
    ).where("row_json IS NOT NULL")
    parsed = rows.select(
        (col("op")=='d').alias("is_delete"),
        col("op"), col("event_ts"),
        from_json(col("row_json"), after_schema).alias("r")
    ).selectExpr("is_delete", "op", "event_ts", "r.*")
    return parsed

# Target SCD2 table with required SCD2 interval columns
dlt.create_streaming_table(
    name="users_scd2",
    table_properties={"delta.enableChangeDataFeed": "true"},
    schema="""
      id INT, name STRING, email STRING, created_at STRING,
      __START_AT TIMESTAMP, __END_AT TIMESTAMP
    """
)

# DLT bitemporal SCD2 flow (AUTO CDC replaces apply_changes; same signature)
dlt.create_auto_cdc_flow(
    target="LIVE.users_scd2",
    source="LIVE.cdc_rows",
    keys=["id"],
    sequence_by="event_ts",
    apply_as_deletes="is_delete = true",
    stored_as_scd_type=2,
    except_column_list=["is_delete", "op", "event_ts"]
)
```

- `create_auto_cdc_flow` is the Python API that supersedes `apply_changes()` (same signature). ([Microsoft Learn](https://learn.microsoft.com/en-us/azure/databricks/dlt-ref/dlt-python-ref-apply-changes?utm_source=chatgpt.com "create_auto_cdc_flow - Azure Databricks"), [Databricks Documentation](https://docs.databricks.com/aws/en/dlt/cdc?utm_source=chatgpt.com "The AUTO CDC APIs: Simplify change data capture with ..."))
    
- `AUTO CDC` / `APPLY CHANGES` are designed for CDC and SCD1/2, handling ordering & late data. ([Databricks](https://www.databricks.com/blog/2022/04/25/simplifying-change-data-capture-with-databricks-delta-live-tables.html?utm_source=chatgpt.com "Change Data Capture With Delta Live Tables"), [Medium](https://medium.com/%40soumen.nandi01/enhancing-databricks-dlt-pipeline-performance-when-using-apply-changes-ad8fa00fb2c3?utm_source=chatgpt.com "Enhancing Databricks DLT Pipeline Performance when ..."))
    

---

## Why this is “bitemporal”

- **Valid time:** DLT SCD2 keeps one row version per key with `__START_AT`..`__END_AT` along your chosen `SEQUENCE BY` (`event_ts` → Debezium `ts_ms`). ([Databricks Documentation](https://docs.databricks.com/aws/en/dlt-ref/dlt-sql-ref-apply-changes-into?utm_source=chatgpt.com "AUTO CDC INTO (Lakeflow Declarative Pipelines)"))
    
- **System time:** Delta table **time travel/history** tracks when each version was committed. Query older **versions** to traverse the system timeline. ([Microsoft Learn](https://learn.microsoft.com/en-us/azure/databricks/delta/history?utm_source=chatgpt.com "Work with Delta Lake table history - Azure Databricks"), [Databricks](https://www.databricks.com/blog/2019/02/04/introducing-delta-time-travel-for-large-scale-data-lakes.html?utm_source=chatgpt.com "Introducing Delta Time Travel for Large Scale Data Lakes"))
    

---

## Notes / gotchas

- Your source into `AUTO CDC` must be **streaming / append-only** (DLT expects this). ([Stack Overflow](https://stackoverflow.com/questions/78142381/databricks-dlt-and-cdc-when-underlying-data-changed?utm_source=chatgpt.com "Databricks DLT and CDC When Underlying Data Changed"))
    
- If events can arrive out of order, `AUTO CDC` is **preferred over manual MERGE**. ([Medium](https://medium.com/%40luijk.r/merge-vs-apply-changes-in-databricks-e9a88dfc9062?utm_source=chatgpt.com "Merge vs. Apply Changes in Databricks | by René Luijk"))
    
- Enable CDF on the target (done above) if downstream needs row-level change logs. ([Databricks Documentation](https://docs.databricks.com/aws/en/delta/delta-change-data-feed?utm_source=chatgpt.com "Use Delta Lake change data feed on Databricks"))
    

If you want, I can paste a minimal **pipeline JSON** you can drop into the DLT UI (with your catalog/schema as the target) so this runs with zero extra plumbing.