---

---
---------------


Perfect — here’s a **clean, production-style `foreachBatch` backfill** that:

- uses your **`date`** column as the event timestamp (contains date+time),
    
- **creates a full 30-minute grid per (tradingDate, ticker, tenor)** so every key has the **same number of snapshots per day**,
    
- **forward-fills only when a value is NULL** (never overwrites non-nulls),
    
- handles duplicate rows within the same 30-min slot (keeps the **latest** by time),
    
- touches **only the days present** in the micro-batch (idempotent & fast),
    
- avoids cross-day carry (no previous-day seeding),
    
- and writes to a **Gold** table.
    

---

# Drop-in code (Databricks / PySpark)

> Replace catalog/table names and the `checkpoint` path to your environment.  
> If your column names differ, adjust `KEY_TICKER`, `KEY_TENOR`, `TIME_COL`.  
> If you want to explicitly list your volatility columns, set `VOL_COLS_FIXED`.

```python
from pyspark.sql import functions as F, Window as W
from pyspark.sql.utils import AnalysisException

# ============================
# CONFIG — edit these
# ============================
CATALOG   = "d4001-centralus-tdvip-tdsbi_catalog"
SILVER_TB = f"`{CATALOG}`.silver.optionsfo_silver"
GOLD_TB   = f"`{CATALOG}`.gold.optionsfo_filled"

KEY_TICKER = "ticker_tk"   # symbol column (change if your schema uses a different name)
KEY_TENOR  = "days"        # tenor column (as in your screenshots)
TIME_COL   = "date"        # your event time column (date+time in a single field)

# If you want to hardcode vol columns, put them here and set VOL_COLS_FIXED != None
VOL_COLS_FIXED = None
# VOL_COLS_FIXED = ["volD10","volD20","volD30","volD40","volATM","vWidth","volU10","volU20","volU30","volU40"]

CHECKPOINT_PATH = "/Volumes/your_volume/_chk/optionsfo_filled"  # <-- change

# ============================
# HELPERS
# ============================
def _floor_to_30min(colname: str):
    secs = 30 * 60
    return F.from_unixtime(F.floor(F.col(colname).cast("long") / secs) * secs).cast("timestamp")

def _safe_get_vol_cols(df):
    """
    Auto-detect volatility columns if not fixed.
    Rule: columns that start with 'vol' (case-insensitive) or exactly 'vWidth'
    """
    if VOL_COLS_FIXED is not None:
        # keep only those that truly exist on DF
        cols = [c for c in VOL_COLS_FIXED if c in df.columns]
        return cols

    lower_map = {c.lower(): c for c in df.columns}
    out = []
    for lc, orig in lower_map.items():
        if lc.startswith("vol") or lc == "vwidth":
            out.append(orig)
    return out

# ============================
# foreachBatch
# ============================
def backfill_microbatch(mb_df, batch_id: int):
    try:
        # 0) Normalize time & filter session on microbatch
        df = (
            mb_df
            .filter(F.col("tradingSession") == "RegularMkt")
            .withColumn("event_ts", F.col(TIME_COL).cast("timestamp"))
            .withColumn("tradingDate", F.to_date("event_ts"))
            .withColumn("slot_ts", _floor_to_30min("event_ts"))
        )

        # If microbatch has no rows after filters, stop quickly
        dates = [r["tradingDate"] for r in df.select("tradingDate").distinct().collect()]
        if not dates:
            return

        # 1) Re-read full SILVER for those dates (idempotent recompute)
        base = (
            spark.read.table(SILVER_TB)
            .filter(F.col("tradingSession") == "RegularMkt")
            .withColumn("event_ts", F.col(TIME_COL).cast("timestamp"))
            .withColumn("tradingDate", F.to_date("event_ts"))
            .withColumn("slot_ts", _floor_to_30min("event_ts"))
            .filter(F.col("tradingDate").isin(dates))
        )

        # If nothing in base (rare), return
        if base.head(1) == []:
            return

        # 2) Work out which vol columns to fill
        VOL_COLS = _safe_get_vol_cols(base)
        if not VOL_COLS:
            # No volatility columns found; nothing to fill
            return

        # 3) Reduce to the columns we actually need for the rest of the steps
        keep_cols = ["tradingDate", KEY_TICKER, KEY_TENOR, "event_ts", "slot_ts"] + VOL_COLS
        base = base.select(*[c for c in keep_cols if c in base.columns])

        # Guard: ensure key columns exist
        for req in ["tradingDate", "slot_ts", KEY_TICKER, KEY_TENOR]:
            if req not in base.columns:
                raise AnalysisException(f"Required column missing: {req}")

        # 4) Deduplicate within the same 30-min slot (keep the latest by event_ts)
        w_slot = W.partitionBy("tradingDate", KEY_TICKER, KEY_TENOR, "slot_ts").orderBy(F.col("event_ts").desc())
        latest_in_slot = (
            base
            .withColumn("_rn", F.row_number().over(w_slot))
            .filter(F.col("_rn") == 1)
            .drop("_rn")
        )

        # 5) Build a uniform 30-min grid per (tradingDate, ticker, tenor)
        #    a) min/max per tradingDate (so we don't assume market hours)
        bounds = (
            latest_in_slot.groupBy("tradingDate")
            .agg(F.min("slot_ts").alias("min_ts"), F.max("slot_ts").alias("max_ts"))
        )

        # Case: if min == max for a date, sequence still gives one slot
        grid = (
            bounds
            .withColumn("slot_ts", F.explode(F.expr("sequence(min_ts, max_ts, interval 30 minutes)")))
            .select("tradingDate", "slot_ts")
        )

        # If grid is empty (shouldn't happen), bail
        if grid.head(1) == []:
            return

        keys = latest_in_slot.select("tradingDate", KEY_TICKER, KEY_TENOR).distinct()

        full_grid = (
            keys.join(grid, "tradingDate", "inner")
                .select("tradingDate", KEY_TICKER, KEY_TENOR, "slot_ts")
        )

        # 6) Join grid ⟵ latest snapshots
        joined = full_grid.join(
            latest_in_slot.select("tradingDate", KEY_TICKER, KEY_TENOR, "slot_ts", *VOL_COLS),
            ["tradingDate", KEY_TICKER, KEY_TENOR, "slot_ts"],
            "left"
        )

        # 7) Forward-fill only when NULL (within the day)
        w_ff = (
            W.partitionBy("tradingDate", KEY_TICKER, KEY_TENOR)
             .orderBy("slot_ts")
             .rowsBetween(W.unboundedPreceding, 0)
        )

        filled = joined
        for c in VOL_COLS:
            last_non_null = F.last(F.when(F.col(c).isNotNull, F.col(c)), ignorenulls=True).over(w_ff)
            filled = filled.withColumn(f"{c}_bf", F.coalesce(F.col(c), last_non_null))

        # 8) Write Gold — overwrite only the changed (date,ticker,tenor) partitions
        #    We partition by (tradingDate, ticker, days) to make day-based overwrites trivial
        spark.conf.set("spark.sql.sources.partitionOverwriteMode", "dynamic")

        (
            filled
            .repartition("tradingDate", KEY_TICKER, KEY_TENOR)
            .write
            .format("delta")
            .mode("overwrite")
            .partitionBy("tradingDate", KEY_TICKER, KEY_TENOR)
            .saveAsTable(GOLD_TB)
        )

    except Exception as e:
        # Let Databricks retry the batch, but surface a clean error message
        # (Re-raise so structured streaming is aware and can handle according to its restart policy.)
        raise

# ============================
# Start the stream
# ============================
(
    spark.readStream.table(SILVER_TB)
    .writeStream
    .foreachBatch(backfill_microbatch)
    .outputMode("update")  # we control actual writes inside foreachBatch
    .option("checkpointLocation", CHECKPOINT_PATH)
    .start()
)
```

---

## What this does (step-by-step, simple)

1. **Read only RegularMkt rows** from the micro-batch; derive:
    
    - `tradingDate = to_date(date)`
        
    - `slot_ts = floor(date to 30-min)`
        
2. **Re-read Silver for just those `tradingDate`s** to make the batch **idempotent** (so replays/late data are fine).
    
3. **Find volatility columns** automatically (anything starting with `vol*` or `vWidth`), or use your fixed list.
    
4. **Deduplicate same-slot rows** (if multiple arrived within the same 30 minutes) by keeping the **latest** (`row_number` over `event_ts desc`).
    
5. For each **day**, build a **30-min time grid** from `min(slot_ts)` to `max(slot_ts)`; cross with all `(ticker, days)` keys seen that day → this guarantees **every key has the same number of rows**.
    
6. **Left join** the grid with your latest-in-slot snapshots → missing slots become **NULL**.
    
7. **Forward-fill only where NULL**: for each `vol*` column, compute the **last non-null so far** in time order and **coalesce(original, last_non_null)**.  
    → real values stay; only gaps (nulls) are filled.
    
8. **Write to Gold**, **partitioned by `(tradingDate, ticker, days)`**, with **dynamic partition overwrite** → only the affected days/keys are replaced.
    

---

## Outcome

- A **Gold** table where each `(tradingDate, ticker_tk, days)` has a **uniform 30-min series** with **no null gaps** in the `_bf` (backfilled) columns.
    
- **Only-null** filling: if a volatility value exists, it remains untouched.
    
- **No cross-day carry**: filling never reads yesterday.
    
- Robust to duplicates, late arrivals, and replays.
    

---

## Quick validations

```sql
-- 1) Every key has uniform slot counts for a day
SELECT tradingDate, ticker_tk, days, COUNT(*) AS slots
FROM   {GOLD_TB}
WHERE  tradingDate = DATE '2025-08-23'
GROUP BY 1,2,3
ORDER BY 1,2,3;

-- 2) Fill-only-when-null check for a sample key
SELECT slot_ts, volD40, volD40_bf
FROM   {GOLD_TB}
WHERE  tradingDate = DATE '2025-08-23'
  AND  ticker_tk = 'SPY'
  AND  days = 10
ORDER BY slot_ts;
```

---

### Notes / Options

- If you already know your exact set of `vol*` columns, define them in `VOL_COLS_FIXED` for faster schema checks.
    
- If you prefer a **fixed market schedule** (e.g., 09:30–16:00) instead of `min→max`, you can replace step (5) to generate a constant per-day `sequence()` for those hours.
    
- You can **Liquid Cluster / Z-ORDER** Gold on `tradingDate, slot_ts` to speed up time-slices for dashboards.
    

This version stays simple, avoids cross-day logic, and focuses exactly on your requirement: **same snapshots per day per ticker/tenor, fill only where null.**


----------------------------

    

> Replace the catalog/table names and checkpoint path to match your workspace.  
> If your symbol/tenor/timestamp columns are named differently, edit `KEY_TICKER`, `KEY_TENOR`, `TIME_COL`.

```python
# Databricks / PySpark

from pyspark.sql import functions as F, Window as W

# ============================
# CONFIG — edit these
# ============================
CATALOG   = "d4001-centralus-tdvip-tdsbi_catalog"
SILVER_TB = f"`{CATALOG}`.silver.optionsfo_silver"
GOLD_TB   = f"`{CATALOG}`.gold.optionsfo_filled"

KEY_TICKER = "ticker_tk"    # change if your symbol column is named differently
KEY_TENOR  = "days"         # tenor column (integer “days” visible in your screenshots)
TIME_COL   = "date"         # your event time column (contains date+time)

# 🔒 Hard-code exactly the volatility fields to backfill
VOL_COLS = [
    "volD10", "volD20", "volD30", "volD40", "volD50", "volD60",
    "volATM", "vWidth",
    # add/remove any you need — only these will be filled
]

CHECKPOINT_PATH = "/Volumes/your_volume/_chk/optionsfo_filled"  # <-- change

# ============================
# HELPERS
# ============================
def _floor_to_30min(colname: str):
    secs = 30 * 60
    return F.from_unixtime(F.floor(F.col(colname).cast("long") / secs) * secs).cast("timestamp")

def _require_columns(df, cols):
    missing = [c for c in cols if c not in df.columns]
    if missing:
        raise Exception(f"Missing required column(s): {missing}")

# ============================
# foreachBatch
# ============================
def backfill_microbatch(mb_df, batch_id: int):
    # 0) Normalize time & filter current micro-batch to RegularMkt
    df = (
        mb_df
        .filter(F.col("tradingSession") == "RegularMkt")
        .withColumn("event_ts", F.col(TIME_COL).cast("timestamp"))
        .withColumn("tradingDate", F.to_date("event_ts"))
        .withColumn("slot_ts", _floor_to_30min("event_ts"))
    )

    # Identify affected trading dates to keep recompute small & idempotent
    dates = [r["tradingDate"] for r in df.select("tradingDate").distinct().collect()]
    if not dates:
        return

    # 1) Re-read Silver for ONLY those dates (so late/replayed data recomputes cleanly)
    base = (
        spark.read.table(SILVER_TB)
        .filter(F.col("tradingSession") == "RegularMkt")
        .withColumn("event_ts", F.col(TIME_COL).cast("timestamp"))
        .withColumn("tradingDate", F.to_date("event_ts"))
        .withColumn("slot_ts", _floor_to_30min("event_ts"))
        .filter(F.col("tradingDate").isin(dates))
    )

    if base.head(1) == []:
        return

    # 2) Guardrails: required columns must exist
    _require_columns(base, ["tradingDate", "slot_ts", KEY_TICKER, KEY_TENOR, "event_ts"])
    # Only keep the columns we actually need
    keep_cols = ["tradingDate", KEY_TICKER, KEY_TENOR, "event_ts", "slot_ts"] + VOL_COLS
    base = base.select(*[c for c in keep_cols if c in base.columns])

    # 3) Deduplicate rows within the same 30-min slot (keep latest by event_ts)
    w_slot = W.partitionBy("tradingDate", KEY_TICKER, KEY_TENOR, "slot_ts").orderBy(F.col("event_ts").desc())
    latest_in_slot = (
        base.withColumn("_rn", F.row_number().over(w_slot))
            .filter(F.col("_rn") == 1)
            .drop("_rn")
    )

    if latest_in_slot.head(1) == []:
        return

    # 4) Build a uniform 30-min grid per day from min→max (no hardcoded market hours)
    bounds = (
        latest_in_slot.groupBy("tradingDate")
        .agg(F.min("slot_ts").alias("min_ts"), F.max("slot_ts").alias("max_ts"))
    )
    grid = (
        bounds
        .withColumn("slot_ts", F.explode(F.expr("sequence(min_ts, max_ts, interval 30 minutes)")))
        .select("tradingDate", "slot_ts")
    )
    if grid.head(1) == []:
        return

    keys = latest_in_slot.select("tradingDate", KEY_TICKER, KEY_TENOR).distinct()

    full_grid = (
        keys.join(grid, "tradingDate", "inner")
            .select("tradingDate", KEY_TICKER, KEY_TENOR, "slot_ts")
    )

    # 5) Join grid ⟵ latest snapshots (produces NULLs for missing slots)
    joined = full_grid.join(
        latest_in_slot.select("tradingDate", KEY_TICKER, KEY_TENOR, "slot_ts", *VOL_COLS),
        ["tradingDate", KEY_TICKER, KEY_TENOR, "slot_ts"],
        "left"
    )

    # 6) Forward-fill only when NULL (stay within the day; no cross-day seeding)
    w_ff = (
        W.partitionBy("tradingDate", KEY_TICKER, KEY_TENOR)
         .orderBy("slot_ts")
         .rowsBetween(W.unboundedPreceding, 0)
    )

    filled = joined
    for c in VOL_COLS:
        # last non-null so far; coalesce keeps true (non-null) values, fills only the nulls
        last_non_null = F.last(F.when(F.col(c).isNotNull, F.col(c)), ignorenulls=True).over(w_ff)
        filled = filled.withColumn(f"{c}_bf", F.coalesce(F.col(c), last_non_null))

    # 7) Write Gold — overwrite only changed (tradingDate, ticker, days) partitions
    spark.conf.set("spark.sql.sources.partitionOverwriteMode", "dynamic")
    (
        filled
        .repartition("tradingDate", KEY_TICKER, KEY_TENOR)  # good shuffle partitioning for targeted overwrite
        .write
        .format("delta")
        .mode("overwrite")
        .partitionBy("tradingDate", KEY_TICKER, KEY_TENOR)
        .saveAsTable(GOLD_TB)
    )

# ============================
# Start the stream
# ============================
(
    spark.readStream.table(SILVER_TB)
    .writeStream
    .foreachBatch(backfill_microbatch)
    .outputMode("update")  # the batch writer controls final writes
    .option("checkpointLocation", CHECKPOINT_PATH)
    .start()
)
```

## Why this meets your exact requirement

- **Same number of snapshots/day per key**: we build a **30-min grid** from `min(slot_ts)`→`max(slot_ts)` for each **tradingDate** and **cross it** with all `(ticker_tk, days)` keys seen that day. That forces a uniform row count across keys.
    
- **Only fill when null**: for every hard-coded vol field in `VOL_COLS`, we compute the **last non-null so far** and `COALESCE(original, last_non_null)`. Real non-nulls stay untouched; gaps get filled.
    
- **Duplicates per slot**: handled by keeping the **latest** row in that slot (`row_number` over `event_ts desc`).
    
- **No cross-day carry**: the window partitions by `tradingDate`, so filling never reads the previous day.
    
- **Idempotent daily recompute**: each micro-batch recomputes **only the affected trading dates** from Silver and **overwrites those partitions** in Gold (dynamic partition overwrite). Late data or replays won’t corrupt results.
    

## Quick validations

```sql
-- 1) Uniform slot counts per key for a day
SELECT tradingDate, ticker_tk, days, COUNT(*) AS slots
FROM   d4001-centralus-tdvip-tdsbi_catalog.gold.optionsfo_filled
WHERE  tradingDate = DATE '2025-08-23'
GROUP BY 1,2,3
ORDER BY 1,2,3;

-- 2) “fill only when null” check
SELECT slot_ts, volD40, volD40_bf
FROM   d4001-centralus-tdvip-tdsbi_catalog.gold.optionsfo_filled
WHERE  tradingDate = DATE '2025-08-23'
  AND  ticker_tk = 'SPY'
  AND  days = 10
ORDER BY slot_ts;
```

If you want me to pre-populate `VOL_COLS` with **your exact names** (from your table schema), paste them and I’ll drop them in so you can run it verbatim.