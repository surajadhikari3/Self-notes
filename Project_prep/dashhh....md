

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


----------------------

Great — your `application-dev.yml` is already set up to send **objects** as JSON:

- `key.serializer = StringSerializer` ✅
    
- `value.serializer` is a JSON serializer (via your delegate `TdSecuredKafkaJsonSerializer`) ✅
    

So the only thing that causes the ugly `\"`/`\\` in Databricks is **double‑serializing in your producer code**.

## What to change (code)

1. Use `KafkaTemplate<String, Object>` (not `<String, String>`).
    
2. **Do not** call `writeValueAsString(...)` on payloads that are already objects/Maps/JsonNodes.
    
3. Send `Map`/`POJO`/`JsonNode` directly; your JSON serializer will turn it into bytes exactly once.
    

```java
@Service
public class KafkaProducerSvc {

  private final KafkaTemplate<String, Object> kafkaTemplate;
  private final ObjectMapper om = new ObjectMapper();

  @Value("${spring.kafka.producer.topic}")   // or your ${topic:poc_only}
  private String topic;

  public KafkaProducerSvc(KafkaTemplate<String, Object> kafkaTemplate) {
    this.kafkaTemplate = kafkaTemplate;
  }

  // File contains an array of JSON objects: [ { ... }, { ... } ]
  public void produceFromFile(String filepath) throws IOException {
    List<Map<String, Object>> records =
        om.readValue(new File(filepath), new com.fasterxml.jackson.core.type.TypeReference<List<Map<String, Object>>>(){});

    for (Map<String, Object> rec : records) {
      String key = String.valueOf(rec.getOrDefault("source", "unknown"));
      // ✅ send the object; your Json serializer handles serialization once
      kafkaTemplate.send(topic, key, rec);
    }
  }

  // If your file has newline-delimited JSON, read as JsonNode and send:
  public void produceLineByLine(Path path) throws IOException {
    try (var lines = java.nio.file.Files.lines(path)) {
      lines.filter(l -> !l.isBlank()).forEach(l -> {
        try {
          com.fasterxml.jackson.databind.JsonNode node = om.readTree(l); // parse to object
          String key = node.path("source").asText("unknown");
          kafkaTemplate.send(topic, key, node); // ✅ send node, not String
        } catch (Exception e) { /* handle */ }
      });
    }
  }
}
```

## If you must keep `<String, String>`

Only serialize **once**:

```java
KafkaTemplate<String, String> template; // value.serializer = StringSerializer

String json = om.writeValueAsString(rec); // ONE time
template.send(topic, key, json);          // do NOT wrap json in quotes or serialize again
```

## Minor YAML tweaks (optional but nice)

If you ever switch to Spring’s `JsonSerializer`, add:

```yaml
spring:
  kafka:
    producer:
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      properties:
        spring.json.add.type.headers: false
```

Your TLS/truststore bits look fine; keeping them under the shared `spring.kafka.properties` block (as you have) is good.

With the code change above, the payload will land in Databricks as clean JSON (`{"a":"b"}`), so your `from_json(value, schema)` will parse without any unescaping step.

--------------------------------------------------------


Awesome—here are two ready‑to‑use snippets. They build one **Silver “Position Data Product”** table with exactly the **7 fields**. All fields come from **position**, except **`cusip`**, which comes from **instrument** after the join.

---

# Option A — Plain Spark (batch/stream‑compatible)

```python
# ASSUMPTIONS: you already have these 6 Bronze (or early Silver) tables/views:
#   bronze_ged_position, bronze_ged_instrument
#   bronze_fi_position,  bronze_fi_instrument
#   bronze_cmdty_position, bronze_cmdty_instrument

from pyspark.sql.functions import col

# --- 1) Conform per source (rename to canonical 7 names) ---

ged_pos = (
  spark.table("bronze_ged_position")
       .select(
         col("allotment").alias("allotment"),
         col("instrumentCode").alias("instrumentCode"),
         col("id").alias("positionId"),
         col("P&L").alias("pnl"),
         col("mtm").alias("mtm"),
         col("isin").alias("isin")
       )
)

fi_pos = (
  spark.table("bronze_fi_position")
       .select(
         col("allotment").alias("allotment"),
         col("instrCode").alias("instrumentCode"),
         col("posId").alias("positionId"),
         col("pnl").alias("pnl"),
         col("mtm").alias("mtm"),
         col("isin").alias("isin")
       )
)

cmdty_pos = (
  spark.table("bronze_cmdty_position")
       .select(
         col("allotment").alias("allotment"),
         col("instrumentCode").alias("instrumentCode"),
         col("positionId").alias("positionId"),
         col("pnl").alias("pnl"),
         col("markToMarket").alias("mtm"),
         col("ISIN").alias("isin")
       )
)

# Instruments (note: GED has nested identifier.cusip, others differ only by case)
ged_instr   = spark.table("bronze_ged_instrument") \
                   .select(col("code").alias("instrumentCode"),
                           col("identifier.cusip").alias("cusip"),
                           col("isin").alias("isin_instr"))  # optional
fi_instr    = spark.table("bronze_fi_instrument") \
                   .select(col("code").alias("instrumentCode"),
                           col("cusip").alias("cusip"))
cmdty_instr = spark.table("bronze_cmdty_instrument") \
                   .select(col("code").alias("instrumentCode"),
                           col("CUSIP").alias("cusip"))

# --- 2) Union positions by name ---
all_pos = ged_pos.unionByName(fi_pos).unionByName(cmdty_pos)

# --- 3) Union instruments by name (dedupe if needed) ---
from pyspark.sql.functions import row_number
from pyspark.sql.window import Window

all_instr = ged_instr.unionByName(fi_instr, allowMissingColumns=True)\
                     .unionByName(cmdty_instr, allowMissingColumns=True)\
                     .dropDuplicates(["instrumentCode"]) # keep latest if you have a timestamp

# --- 4) Join to enrich positions with CUSIP (ONLY field taken from instrument) ---
position_data_product = (
  all_pos.alias("p")
        .join(all_instr.alias("i"), on="instrumentCode", how="left")
        .select(
            col("p.allotment").alias("allotment"),
            col("p.instrumentCode").alias("instrumentCode"),
            col("p.positionId").alias("positionId"),
            col("p.pnl").alias("pnl"),
            col("p.mtm").alias("mtm"),
            col("p.isin").alias("isin"),
            col("i.cusip").alias("cusip")     # <-- only from instrument
        )
)

# --- 5) Save Silver table ---
position_data_product.write.mode("overwrite").format("delta") \
  .saveAsTable("catalog.schema.silver_position_data_product")
```

---

# Option B — Delta Live Tables (streaming pipeline)

```python
import dlt
from pyspark.sql.functions import col

# === Conform per source ===

@dlt.table
def silver_ged_position_conform():
    s = dlt.read_stream("bronze_ged_position")
    return s.select(
        col("allotment").alias("allotment"),
        col("instrumentCode").alias("instrumentCode"),
        col("id").alias("positionId"),
        col("P&L").alias("pnl"),
        col("mtm").alias("mtm"),
        col("isin").alias("isin")
    )

@dlt.table
def silver_fi_position_conform():
    s = dlt.read_stream("bronze_fi_position")
    return s.select(
        col("allotment").alias("allotment"),
        col("instrCode").alias("instrumentCode"),
        col("posId").alias("positionId"),
        col("pnl").alias("pnl"),
        col("mtm").alias("mtm"),
        col("isin").alias("isin")
    )

@dlt.table
def silver_cmdty_position_conform():
    s = dlt.read_stream("bronze_cmdty_position")
    return s.select(
        col("allotment").alias("allotment"),
        col("instrumentCode").alias("instrumentCode"),
        col("positionId").alias("positionId"),
        col("pnl").alias("pnl"),
        col("markToMarket").alias("mtm"),
        col("ISIN").alias("isin")
    )

@dlt.view
def silver_all_positions_conform():
    return (dlt.read_stream("silver_ged_position_conform")
            .unionByName(dlt.read_stream("silver_fi_position_conform"))
            .unionByName(dlt.read_stream("silver_cmdty_position_conform")))

# Instruments (flatten GED.identifier.cusip)
@dlt.view
def silver_all_instruments_conform():
    ged = dlt.read_stream("bronze_ged_instrument") \
             .select(col("code").alias("instrumentCode"),
                     col("identifier.cusip").alias("cusip"))
    fi = dlt.read_stream("bronze_fi_instrument") \
            .select(col("code").alias("instrumentCode"),
                    col("cusip").alias("cusip"))
    cmdty = dlt.read_stream("bronze_cmdty_instrument") \
              .select(col("code").alias("instrumentCode"),
                      col("CUSIP").alias("cusip"))
    return ged.unionByName(fi, allowMissingColumns=True) \
              .unionByName(cmdty, allowMissingColumns=True)

# FINAL Silver data product (7 fields; cusip only from instrument)
@dlt.table(
  name="silver_position_data_product",
  comment="Position Data Product joined with instrument to fetch CUSIP"
)
def silver_position_data_product():
    p = dlt.read_stream("silver_all_positions_conform").alias("p")
    i = dlt.read_stream("silver_all_instruments_conform").alias("i")
    return (p.join(i, on="instrumentCode", how="left")
             .select(
               col("p.allotment").alias("allotment"),
               col("p.instrumentCode").alias("instrumentCode"),
               col("p.positionId").alias("positionId"),
               col("p.pnl").alias("pnl"),
               col("p.mtm").alias("mtm"),
               col("p.isin").alias("isin"),
               col("i.cusip").alias("cusip")
             ))
```

> This produces a single **Silver** table with the canonical schema:  
> `allotment, instrumentCode, positionId, pnl, mtm, isin, cusip`.  
> Every column is from **position**, **except `cusip`**, which is taken from **instrument** after the join.

If you want, I can add a small SCD‑2 layer for instruments and point the join to `is_current=true` records, plus show how to add Unity Catalog tags (`system_of_origin` on Bronze; `data_product=true`, `cdo_approved=true` on Silver).

-----------------------------

iiiii


Great—use the **business event time** from your JSON (`timeContext.infoSetTime.$date`) as the SCD2 sequence column.

Below is a **drop‑in replacement** for the _instruments conform + SCD2_ part of your DLT. It extracts that nested field (note the backticks around `$date`) and uses it as `_event_ts`. If it’s missing, we fall back to `current_timestamp()` so the pipeline never stalls.

```python
import dlt
from pyspark.sql.functions import col, coalesce, current_timestamp, to_timestamp

# ---------- Instruments → conform & UNION (event time from timeContext.infoSetTime.$date) ----------

@dlt.view
def silver_all_instruments_conform():
    # GED: nested JSON path timeContext.infoSetTime.$date
    ged = dlt.read_stream("bronze_ged_instrument").select(
        col("code").alias("instrumentCode"),
        col("identifier.cusip").alias("cusip"),
        col("isin").alias("isin"),
        # IMPORTANT: backticks around `$date`, then cast to timestamp (UTC string like 2025-08-20T23:59:59.999Z)
        coalesce(
            to_timestamp(col("timeContext.infoSetTime.`$date`")),
            current_timestamp()
        ).alias("_event_ts"),
        col("_source_system") if "_source_system" in dlt.read("bronze_ged_instrument").columns else current_timestamp().alias("_source_system")
    )

    # Fixed Income (adjust if you also get timeContext there; otherwise keep fallback)
    fi = dlt.read_stream("bronze_fi_instrument").select(
        col("code").alias("instrumentCode"),
        col("cusip").alias("cusip"),
        col("isin").alias("isin"),
        coalesce(
            to_timestamp(col("timeContext.infoSetTime.`$date`")),  # keep if present, else fallback
            current_timestamp()
        ).alias("_event_ts"),
        col("_source_system") if "_source_system" in dlt.read("bronze_fi_instrument").columns else current_timestamp().alias("_source_system")
    )

    # Commodities (same idea)
    cmdty = dlt.read_stream("bronze_cmdty_instrument").select(
        col("code").alias("instrumentCode"),
        col("CUSIP").alias("cusip"),
        col("isin").alias("isin"),
        coalesce(
            to_timestamp(col("timeContext.infoSetTime.`$date`")),
            current_timestamp()
        ).alias("_event_ts"),
        col("_source_system") if "_source_system" in dlt.read("bronze_cmdty_instrument").columns else current_timestamp().alias("_source_system")
    )

    return ged.unionByName(fi, allowMissingColumns=True) \
              .unionByName(cmdty, allowMissingColumns=True)

# ---------- SCD2 using the business event time ----------

dlt.create_target_table(
    name="silver_instrument_scd2",
    comment="SCD2 history of instruments (keyed by instrumentCode, sequenced by infoSetTime)"
)

dlt.apply_changes(
    target="silver_instrument_scd2",
    source="silver_all_instruments_conform",
    keys=["instrumentCode"],
    sequence_by=col("_event_ts"),                 # << uses infoSetTime.$date
    stored_as_scd_type=2,
    track_history_columns=["cusip", "isin", "_source_system"]
)

# ---------- Final join stays the same (joins positions to CURRENT instrument) ----------
@dlt.table(
  name="silver_position_data_product",
  comment="Position Data Product (7 fields) joined to current instrument to fetch CUSIP"
)
def silver_position_data_product():
    p = dlt.read_stream("silver_all_positions_conform").alias("p")
    i = (spark.table("LIVE.silver_instrument_scd2")
           .where("__END_AT IS NULL")
           .select(col("instrumentCode"), col("cusip")))
    return (p.join(i, on="instrumentCode", how="left")
             .select(
               col("p.allotment").alias("allotment"),
               col("p.instrumentCode").alias("instrumentCode"),
               col("p.positionId").alias("positionId"),
               col("p.pnl").alias("pnl"),
               col("p.mtm").alias("mtm"),
               col("p.isin").alias("isin"),
               col("i.cusip").alias("cusip")
             ))
```

### Notes

- The key trick is referencing the unusual field name: `col("timeContext.infoSetTime.`$date`")`.
    
- `to_timestamp(...)` will parse the ISO‑8601 string ending with `Z` (UTC). If you prefer, you can be explicit:  
    `to_timestamp(col("timeContext.infoSetTime.`$date`"), "yyyy-MM-dd'T'HH:mm:ss.SSS'Z'")`.
    
- If other sources don’t have `timeContext.infoSetTime.$date`, they’ll fall back to `current_timestamp()`—feel free to replace with their own event-time field if available.
    

If you want, I can also wire in `validFrom/validTo` from your JSON into the SCD2 table for auditing (kept as additional history columns).