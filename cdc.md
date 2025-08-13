---
----------------:
---


import dlt
from pyspark.sql.functions import col, get_json_object, from_json, to_timestamp, coalesce, current_timestamp, length, trim
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
        "d4001-centralus-tdvip-tdsbi_catalog.bronze.debezium_user_events"
    ).selectExpr("CAST(payload AS STRING) AS payload")

@dlt.table
def cdc_rows():
    env = dlt.read_stream("bronze_raw").selectExpr(
        "get_json_object(payload, '$.payload.op') AS op",
        "get_json_object(payload, '$.payload.after')  AS after_json",
        "get_json_object(payload, '$.payload.before') AS before_json"
    ).where("payload IS NOT NULL AND length(trim(payload)) > 0")

    rows = env.selectExpr(
        "CASE WHEN op='d' THEN before_json ELSE after_json END AS row_json",
        "op"
    ).where("row_json IS NOT NULL")

    parsed = rows.select(
        (col("op") == "d").alias("is_delete"),
        col("op"),
        from_json(col("row_json"), after_schema).alias("r")
    ).select(
        "is_delete", "op",
        col("r.id").alias("id"),
        col("r.name").alias("name"),
        col("r.email").alias("email"),
        col("r.created_at").alias("created_at")
    ).withColumn(
        "created_at_ts",
        coalesce(to_timestamp(col("created_at"), "yyyy-MM-dd HH:mm:ss"), current_timestamp())
    )

    # 🔒 Return ONLY these columns. Also drop event_ts if it ever exists.
    return (parsed
            .drop("event_ts")  # safe even if not present
            .select("is_delete", "op", "id", "name", "email", "created_at", "created_at_ts"))
