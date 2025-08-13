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