Great call-out — Debezium can emit **deletes** in two ways:

1. `op='d'` with `payload.before` filled and `payload.after=null`
    
2. **Tombstones** for log compaction, where the **entire value is null** (your `payload` column is null/empty)
    

Use the Connect-safe pipeline below. It:

- ignores true tombstones (value/payload null or empty),
    
- picks `before` for deletes and `after` otherwise,
    
- infers the row schema (or you can plug a fixed schema),
    
- adds an `is_delete` flag.
    

```python
from pyspark.sql.functions import (
    col, get_json_object, from_json, first, schema_of_json,
    when, length, trim, lit
)

# 0) Drop true tombstones (Kafka value null) => payload is null/empty string
non_tombstones = (
    df.where(col("payload").isNotNull() & (length(trim(col("payload"))) > 0))
)

# 1) Pull op/before/after JSON texts from the Debezium envelope
env = non_tombstones.select(
    get_json_object(col("payload"), "$.payload.op").alias("op"),
    get_json_object(col("payload"), "$.payload.after").alias("after_json"),
    get_json_object(col("payload"), "$.payload.before").alias("before_json"),
)

# 2) Choose the correct row JSON: 'before' on delete, else 'after'
row_json_df = env.select(
    when(col("op") == "d", col("before_json")).otherwise(col("after_json")).alias("row_json"),
    (col("op") == "d").alias("is_delete"),
    col("op")
).where(col("row_json").isNotNull())  # still filters out records with neither before nor after

# 3A) Infer the schema of the row object (works on Spark Connect)
schema_ddl = row_json_df.select(schema_of_json(first("row_json", True)).alias("schema")).collect()[0]["schema"]

parsed = (
    row_json_df
      .select(from_json(col("row_json"), schema_ddl).alias("row"), "is_delete", "op")
      .select("is_delete", "op", "row.*")
)

display(parsed)
```

### If you prefer a strict schema (skip inference)

Define it once and replace step 3A with:

```python
from pyspark.sql.types import StructType, StructField, IntegerType, StringType, TimestampType

after_schema = StructType([
    StructField("id", IntegerType(), True),
    StructField("name", StringType(), True),
    StructField("email", StringType(), True),
    StructField("created_at", StringType(), True),   # or TimestampType() if your format matches
    # ...add the rest as per your table
])

parsed = (
    row_json_df
      .select(from_json(col("row_json"), after_schema).alias("row"), "is_delete", "op")
      .select("is_delete", "op", "row.*")
)

display(parsed)
```

This will give you a clean DataFrame you can display in Databricks, with deletes correctly represented and tombstones safely ignored.


------------------


Here’s a **drop-in, streaming-safe** Databricks snippet that:

- Works when `df` (or `payload_df`) is a **streaming** DataFrame
    
- Handles **tombstones** (`payload` null/empty)
    
- Uses **before for deletes**, **after otherwise**
    
- Produces a clean table you can `display()` without the streaming error
    

> Assumes you already have a column named **`payload`** of type **STRING** that contains the Debezium envelope.

```python
# ========================
# Debezium payload → rows
# ========================

from pyspark.sql.functions import (
    col, get_json_object, from_json, when, length, trim
)
from pyspark.sql.types import (
    StructType, StructField, IntegerType, StringType
    # use TimestampType/LongType if you want, but StringType is safest to start
)

# 0) Hardcode the schema of the Debezium "after"/"before" object (your table’s columns)
#    Adjust fields if your table differs.
after_schema = StructType([
    StructField("id", IntegerType(), True),
    StructField("name", StringType(), True),
    StructField("email", StringType(), True),
    StructField("created_at", StringType(), True),
])

# ---- Choose the source DF that has the 'payload' string column ----
# If you created payload_df = df.select(col("payload")), use that:
source = payload_df  # or just use your 'df' if it already has 'payload'

# 1) Drop true tombstones (Kafka value null/empty → payload null/blank)
non_tombstones = source.where(
    col("payload").isNotNull() & (length(trim(col("payload"))) > 0)
)

# 2) Extract Debezium envelope fields (all strings)
env = non_tombstones.select(
    get_json_object(col("payload"), "$.payload.op").alias("op"),
    get_json_object(col("payload"), "$.payload.after").alias("after_json"),
    get_json_object(col("payload"), "$.payload.before").alias("before_json"),
)

# 3) Use 'before' for deletes, else 'after', and keep an is_delete flag
row_json_df = (
    env.select(
        when(col("op") == "d", col("before_json"))
          .otherwise(col("after_json"))
          .alias("row_json"),
        (col("op") == "d").alias("is_delete"),
        col("op")
    )
    .where(col("row_json").isNotNull())
)

# 4) Parse the JSON row into columns (no RDDs, no collect/first → streaming-safe)
parsed = (
    row_json_df
      .withColumn("row", from_json(col("row_json"), after_schema))
      .select("is_delete", "op", "row.*")
)

# ========================
# How to "display" safely
# ========================

# A) In-notebook live view (memory sink). Works great for quick inspection.
#    Then query it with spark.sql(...) as a normal (batch) table.
query = (
    parsed.writeStream
          .format("memory")
          .queryName("debezium_rows")   # creates an in-memory table
          .outputMode("append")
          .start()
)

# Now you can run this in a separate cell to see the data, without streaming errors:
# display(spark.sql("SELECT * FROM debezium_rows"))

# ---- OR ----

# B) Persist to Delta (recommended for real pipelines). Replace the path & checkpoint.
# query = (
#     parsed.writeStream
#           .format("delta")
#           .outputMode("append")
#           .option("checkpointLocation", "/mnt/checkpoints/debezium_rows_ckpt")
#           .start("/mnt/delta/debezium_rows")
# )
# Then read/display with:
# display(spark.read.format("delta").load("/mnt/delta/debezium_rows"))
```

### Notes

- If your table columns differ, just edit `after_schema` (add/remove fields).
    
- Once data is flowing, you can cast `created_at` to `timestamp` if needed:
    
    ```python
    from pyspark.sql.functions import to_timestamp
    clean = spark.sql("select * from debezium_rows") \
                 .withColumn("created_at_ts", to_timestamp(col("created_at")))
    display(clean)
    ```
    
- If you also want the **primary key** on deletes for downstream upserts, make sure it’s present in `before` and included in the schema (e.g., `id`).