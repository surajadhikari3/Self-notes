

Perfect—since your stream has only two logical record types (by **key**): **Instrument** and **Position**, here’s a clean Databricks PySpark pattern that:

1. cleans the escaped JSON `value`,
    
2. classifies each row as `instrument` or `position` from the `key`,
    
3. parses with the right schema, and
    
4. writes each type to its own Delta table in one streaming query.
    

Just paste this into your notebook and update the two table names (and, ideally, the schemas).

```python
from pyspark.sql import functions as F, types as T

# ── 0) Your input stream: must have columns key:string, value:string (Kafka value already cast to string)
# processed_df = <your prepared DataFrame>

# ── 1) Normalize key and clean the JSON text (handles quoted + backslash-escaped payloads)
cleaned = (
    processed_df
    .select(
        F.trim(F.col("key")).alias("key"),
        F.col("value").cast("string").alias("raw_value"),
    )
    # if whole payload is a quoted JSON string, strip the outer quotes:
    .withColumn("json_str", F.regexp_replace(F.col("raw_value"), r'^\s*"(.*)"\s*$', r'\1'))
    # unescape backslashes (turn {\"a\":\"b\"} into {"a":"b"}):
    .withColumn("json_str", F.regexp_replace(F.col("json_str"), r'\\', ''))
    # classify record type from key text
    .withColumn(
        "record_type",
        F.when(F.lower(F.col("key")).contains("position"), F.lit("position"))
         .when(F.lower(F.col("key")).contains("instrument"), F.lit("instrument"))
         .otherwise(F.lit("unknown"))
    )
)

# ── 2) Define schemas (recommended). Start with the fields you need; you can evolve them later.
INSTRUMENT_SCHEMA = T.StructType([
    # Example fields – replace with your actual instrument fields:
    T.StructField("instrument", T.StructType([
        T.StructField("code", T.StringType()),
        T.StructField("productType", T.StringType()),
        T.StructField("feature", T.StringType()),
    ])),
    T.StructField("source", T.StringType()),
    T.StructField("asOfDate", T.StringType()),
    # ... add more as needed ...
])

POSITION_SCHEMA = T.StructType([
    # Example fields – replace with your actual position fields:
    T.StructField("position", T.StructType([
        T.StructField("level", T.StringType()),
        T.StructField("vega", T.DoubleType()),
        T.StructField("gammaNotional", T.DoubleType()),
        # ...
    ])),
    T.StructField("source", T.StringType()),
    T.StructField("asOfDate", T.StringType()),
    # ... add more as needed ...
])

# ── 3) Target tables (Unity Catalog fully qualified names recommended)
INSTRUMENT_TABLE = "your_catalog.your_schema.instrument_delta"
POSITION_TABLE   = "your_catalog.your_schema.position_delta"

# Separate checkpoint folders per sink + a master one for the query
CP_ROOT = "dbfs:/checkpoints/kafka_two_types"

# ── 4) foreachBatch writer: split by type, parse with correct schema, then write
def write_by_type(batch_df, batch_id: int):
    if batch_df.rdd.isEmpty():
        return

    # INSTRUMENT
    inst_df = batch_df.filter(F.col("record_type") == "instrument")
    if not inst_df.rdd.isEmpty():
        parsed_inst = inst_df.withColumn("data", F.from_json("json_str", INSTRUMENT_SCHEMA))
        out_inst = (
            parsed_inst
            .select(
                F.col("key").alias("source_key"),
                F.col("data.*")  # flatten
            )
        )
        (out_inst.write
            .format("delta")
            .mode("append")
            .option("mergeSchema", "true")   # keep on while aligning fields; turn off when stable
            .option("checkpointLocation", f"{CP_ROOT}/instrument")
            .saveAsTable(INSTRUMENT_TABLE)
        )

    # POSITION
    pos_df = batch_df.filter(F.col("record_type") == "position")
    if not pos_df.rdd.isEmpty():
        parsed_pos = pos_df.withColumn("data", F.from_json("json_str", POSITION_SCHEMA))
        out_pos = (
            parsed_pos
            .select(
                F.col("key").alias("source_key"),
                F.col("data.*")
            )
        )
        (out_pos.write
            .format("delta")
            .mode("append")
            .option("mergeSchema", "true")
            .option("checkpointLocation", f"{CP_ROOT}/position")
            .saveAsTable(POSITION_TABLE)
        )

# ── 5) Start one stream that fans out to both tables
query = (
    cleaned.writeStream
           .foreachBatch(write_by_type)
           .outputMode("append")
           .option("checkpointLocation", f"{CP_ROOT}/master_stream")
           .start()
)

# query.awaitTermination()  # uncomment if you want to block the cell
```

### Optional add‑ons

- **Schema inference while prototyping**: If you don’t know the schemas yet, read a small sample to infer:
    
    ```python
    sample_schema = spark.read.json(cleaned.limit(1000).select("json_str").rdd.map(lambda r: r["json_str"])).schema
    ```
    
    Then plug `sample_schema` in place of `INSTRUMENT_SCHEMA` or `POSITION_SCHEMA`.
    
- **Domain column**: If you also want the asset class (e.g., “Equity/Commodity/Fixed Income”) that appears in your `key`, add:
    
    ```python
    cleaned = cleaned.withColumn(
        "domain",
        F.when(F.lower("key").contains("equity"), "Equity")
         .when(F.lower("key").contains("commodity"), "Commodity")
         .when(F.lower("key").contains("fixed income"), "Fixed Income")
         .otherwise("Unknown")
    )
    ```
    
    and keep `domain` in `select(...)` for downstream analytics or partitioning.
    

This keeps the logic simple: classify → clean → parse (per type) → write to two Delta tables, all within a single streaming job.