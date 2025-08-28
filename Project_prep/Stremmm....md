


Perfect — I wired your exact view names + catalog/schema + the three sources (`GED`, `FIXED INCOME`, `COMMODITIES`) into a **ready-to-run Streamlit app** that auto-refreshes and filters by `_source_system`.

---

# 0) Install

```bash
pip install streamlit databricks-sql-connector pandas altair python-dotenv
```

---

# 1) Create `.env`

> Put this next to `app.py`.

```
DATABRICKS_SERVER_HOSTNAME=adb-xxxxxxxx.xx.azuredatabricks.net
DATABRICKS_HTTP_PATH=/sql/1.0/warehouses/xxxxxxxxxxxxxxxx
DATABRICKS_TOKEN=dapiex_xxxxxxxxxxxxxxxxxxxxxxxxxxxxx

CATALOG=d4001-centralus-tdvip-tdsbi_catalog
SCHEMA=gold
```

---

# 2) Save as `app.py`

```python
import os
import streamlit as st
import pandas as pd
import altair as alt
from databricks import sql
from dotenv import load_dotenv

load_dotenv()
st.set_page_config(page_title="DLT Parameterized RT Dashboard", layout="wide")

CATALOG = os.getenv("CATALOG")
SCHEMA  = os.getenv("SCHEMA")

# ---------- Databricks connection ----------
@st.cache_resource
def get_conn():
    return sql.connect(
        server_hostname=os.environ["DATABRICKS_SERVER_HOSTNAME"],
        http_path=os.environ["DATABRICKS_HTTP_PATH"],
        access_token=os.environ["DATABRICKS_TOKEN"],
    )

@st.cache_data(ttl=5, show_spinner=False)
def run_query(q: str, params: tuple = ()):
    with get_conn().cursor() as c:
        c.execute(q, params)
        cols = [d[0] for d in c.description]
        rows = c.fetchall()
    return pd.DataFrame(rows, columns=cols)

# ---------- Sidebar controls ----------
with st.sidebar:
    st.header("Controls")
    # polling interval
    refresh_ms = st.slider("Auto-refresh (ms)", 1000, 30000, 5000, step=1000)

    # fixed set of three sources as requested
    SOURCE_CHOICES = ["GED", "FIXED INCOME", "COMMODITIES"]
    source_system = st.selectbox("_source_system", SOURCE_CHOICES, index=0)

# trigger rerun on a timer
st.autorefresh(interval=refresh_ms, key="auto")

st.title("📊 DLT Parameterized Dashboard (Streamlit, near real-time)")
st.caption(
    f"Catalog: `{CATALOG}` • Schema: `{SCHEMA}` • Source: `{source_system}` • "
    f"Polling every {refresh_ms/1000:.0f}s"
)

# ---------- Queries (YOUR views) ----------
# 1) position data overall (table)
q_overall = f"""
SELECT allotment, instrumentCode, positionId, pnl, mtm, isin, cusip, source_system
FROM {CATALOG}.{SCHEMA}.v_position_overall
WHERE source_system = ?
"""
df_overall = run_query(q_overall, (source_system,))

# 2) position with the highest pnl (single row)
q_highest = f"""
SELECT allotment, instrumentCode, positionId, pnl, mtm, isin, cusip, source_system
FROM {CATALOG}.{SCHEMA}.v_position_with_highest_pnl
WHERE source_system = ?
ORDER BY pnl DESC
LIMIT 1
"""
df_highest = run_query(q_highest, (source_system,))
highest_pnl = float(df_highest["pnl"].iloc[0]) if len(df_highest) else 0.0
highest_instr = df_highest["instrumentCode"].iloc[0] if len(df_highest) else "—"

# 3) sum of pnl by allotment (bar)
q_allot = f"""
SELECT allotment, total_pnl
FROM {CATALOG}.{SCHEMA}.v_sum_pnl_by_allotment
WHERE source_system = ?
ORDER BY total_pnl DESC
"""
df_allot = run_query(q_allot, (source_system,))

# 4) top 10 instruments by pnl (bar)
q_top10 = f"""
SELECT instrumentCode, total_pnl
FROM {CATALOG}.{SCHEMA}.v_top_ten_instruments
WHERE source_system = ?
ORDER BY total_pnl DESC
"""
df_top10 = run_query(q_top10, (source_system,)).head(10)

# ---------- KPIs ----------
k1, k2, k3 = st.columns(3)
k1.metric("Highest P&L", f"${highest_pnl:,.2f}", help=f"Instrument: {highest_instr}")
k2.metric("Rows (overall)", f"{len(df_overall):,}")
k3.metric("Instruments in Top-10 view", f"{len(df_top10):,}")

# ---------- Charts ----------
left, right = st.columns((1.25, 1))

with left:
    st.subheader("Sum of P&L by Allotment")
    if len(df_allot):
        df_allot = df_allot.copy()
        df_allot["sign"] = df_allot["total_pnl"].apply(lambda x: "Gain" if x >= 0 else "Loss")
        chart_allot = (
            alt.Chart(df_allot)
            .mark_bar()
            .encode(
                x=alt.X("total_pnl:Q", title="Total P&L"),
                y=alt.Y("allotment:N", sort="-x", title=""),
                color=alt.Color("sign:N",
                                scale=alt.Scale(domain=["Gain","Loss"], range=["#16a34a","#ef4444"]),
                                legend=None),
                tooltip=[alt.Tooltip("total_pnl:Q", format=",.2f"), "allotment:N"]
            )
            .interactive()
        )
        st.altair_chart(chart_allot, use_container_width=True)
    else:
        st.info("No allotment data for this source.")

with right:
    st.subheader("Top 10 Instruments by P&L")
    if len(df_top10):
        df_top10 = df_top10.copy()
        df_top10["sign"] = df_top10["total_pnl"].apply(lambda x: "Gain" if x >= 0 else "Loss")
        chart_top = (
            alt.Chart(df_top10)
            .mark_bar()
            .encode(
                x=alt.X("total_pnl:Q", title="Total P&L"),
                y=alt.Y("instrumentCode:N", sort="-x", title=""),
                color=alt.Color("sign:N",
                                scale=alt.Scale(domain=["Gain","Loss"], range=["#16a34a","#ef4444"]),
                                legend=None),
                tooltip=[alt.Tooltip("total_pnl:Q", format=",.2f"), "instrumentCode:N"]
            )
            .interactive()
        )
        st.altair_chart(chart_top, use_container_width=True)
    else:
        st.info("No top 10 data for this source.")

# ---------- Detail table ----------
st.subheader("Position Data (overall)")
if len(df_overall):
    st.dataframe(
        df_overall.sort_values("pnl", ascending=False),
        use_container_width=True,
        height=420
    )
else:
    st.info("No rows for this source in v_position_overall.")
```

---

# 3) Run it

```bash
streamlit run app.py
```

That’s it. The page polls your **Gold materialized views** through a **Databricks SQL Warehouse**, auto-refreshes on the interval you set (default 5s), and keeps your `_source_system` parameter locked to the three sources you mentioned.

---

## Notes (to avoid hiccups)

- The code expects each view to expose a **`source_system`** column with values exactly `GED`, `FIXED INCOME`, `COMMODITIES`.
    
- Tweak the **`ttl=5`** on `@st.cache_data` and the **refresh slider** to balance freshness vs. warehouse QPS/cost.
    
- If your column names differ (e.g., `instrumentcode` lower-case, or `total_pnl` named differently), just change the SELECTs accordingly.
    
- This is **polling** (near real-time). For sub-second “ticks”, you’d need a push/WebSocket layer, but 1–5s works great for ops dashboards.

------------------------------------------


You’re rendering the title/caption but none of the tiles—so the app is likely **stuck while querying** (bad env vars / wrong view names / no SQL Warehouse) and there’s no error surfacing yet.

Drop in this **debug-friendly version**. It shows a live status panel, verifies env vars, does a `SELECT 1` smoke test, and surfaces any Databricks errors right in the UI.

Replace your connection + sidebar + first queries with this:

```python
import os, pandas as pd, streamlit as st
from pathlib import Path
from databricks import sql
from dotenv import load_dotenv

# ----- load .env from same folder as app.py -----
load_dotenv(dotenv_path=Path(__file__).with_name(".env"))

CATALOG = os.getenv("CATALOG")
SCHEMA  = os.getenv("SCHEMA")
HOST    = os.getenv("DATABRICKS_SERVER_HOSTNAME")
HPATH   = os.getenv("DATABRICKS_HTTP_PATH")
TOKEN   = os.getenv("DATABRICKS_TOKEN")

st.set_page_config(page_title="DLT Parameterized Dashboard (Streamlit)", layout="wide")

# ----- helpers -----
def mask(s: str, keep=6):
    if not s: return "MISSING"
    return s[:keep] + "…" if len(s) > keep else s

@st.cache_resource
def get_conn():
    # raises if hostname/http_path/token are wrong
    return sql.connect(server_hostname=HOST, http_path=HPATH, access_token=TOKEN)

@st.cache_data(ttl=5, show_spinner=False)
def run_query(q: str, params: tuple = ()):
    with get_conn().cursor() as c:
        c.execute(q, params)
        cols = [d[0] for d in c.description]
        rows = c.fetchall()
    return pd.DataFrame(rows, columns=cols)

# ----- Sidebar controls -----
with st.sidebar:
    st.header("Controls")
    refresh_ms = st.slider("Auto-refresh (ms)", 1000, 30000, 3000, 1000)

    # Connection / env quick check
    with st.expander("Connection status", expanded=True):
        st.write("Host:", mask(HOST))
        st.write("HTTP path:", mask(HPATH))
        st.write("Token:", mask(TOKEN))
        st.write("Catalog/Schema:", f"{CATALOG}.{SCHEMA}" if CATALOG and SCHEMA else "MISSING")

        test = st.button("Test connection")
        if test:
            try:
                df = run_query("SELECT 1 AS ok")
                st.success(f"SQL ok → {df.iloc[0,0]}")
            except Exception as e:
                st.exception(e)

# ---- auto refresh (via helper pkg) ----
from streamlit_autorefresh import st_autorefresh
st_autorefresh(interval=refresh_ms, key="auto")

st.title("📊 DLT Parameterized Dashboard (Streamlit, near real-time)")
st.caption(f"Catalog: {CATALOG} • Schema: {SCHEMA} • Polling every {refresh_ms/1000:.0f}s")

# ----- guardrails -----
missing = [k for k,v in {"HOST":HOST,"HPATH":HPATH,"TOKEN":TOKEN,"CATALOG":CATALOG,"SCHEMA":SCHEMA}.items() if not v]
if missing:
    st.error(f"Missing env vars: {', '.join(missing)}. Put them in a .env beside app.py.")
    st.stop()

# ----- data source_system values -----
try:
    with st.status("Loading source_system choices…", expanded=False) as s:
        ss_df = run_query(f"SELECT DISTINCT source_system FROM {CATALOG}.{SCHEMA}.v_position_overall ORDER BY 1")
        if ss_df.empty:
            s.update(label="No rows from v_position_overall", state="error")
            st.stop()
        SOURCE_CHOICES = ss_df["source_system"].tolist()
        s.update(label="Loaded", state="complete")
except Exception as e:
    st.exception(e)
    st.stop()

source_system = st.selectbox("source_system", SOURCE_CHOICES)

# ----- queries (each wrapped to show errors) -----
def safe_query(label, sql_text, params):
    with st.status(label, expanded=False) as s:
        try:
            df = run_query(sql_text, params)
            s.update(label=f"{label}: {len(df)} rows", state="complete")
            return df
        except Exception as e:
            s.update(label=f"{label}: error", state="error")
            st.exception(e)
            return pd.DataFrame()

q_overall = f"""
SELECT allotment, instrumentCode, positionId, pnl, mtm, isin, cusip, source_system
FROM {CATALOG}.{SCHEMA}.v_position_overall
WHERE source_system = ?
"""

q_highest = f"""
SELECT allotment, instrumentCode, positionId, pnl, mtm, isin, cusip, source_system
FROM {CATALOG}.{SCHEMA}.v_position_with_highest_pnl
WHERE source_system = ?
ORDER BY pnl DESC
LIMIT 1
"""

q_allot = f"""
SELECT allotment, total_pnl
FROM {CATALOG}.{SCHEMA}.v_sum_pnl_by_allotment
WHERE source_system = ?
ORDER BY total_pnl DESC
"""

q_top10 = f"""
SELECT instrumentCode, total_pnl
FROM {CATALOG}.{SCHEMA}.v_top_ten_instruments
WHERE source_system = ?
ORDER BY total_pnl DESC
"""

df_overall = safe_query("Query: overall", q_overall, (source_system,))
df_highest = safe_query("Query: highest pnl", q_highest, (source_system,))
df_allot   = safe_query("Query: sum pnl by allotment", q_allot, (source_system,))
df_top10   = safe_query("Query: top 10 instruments", q_top10, (source_system,))
```

### What this will tell you

- If **env vars aren’t loaded**, you’ll get a red message immediately.
    
- If **connection fails** (bad host/http path/token or no warehouse running), you’ll see the exact exception.
    
- If **view names are wrong** (e.g., `v_position_overall` vs your actual names) you’ll see an error for that specific query.
    
- If the queries return **0 rows**, the status shows “0 rows” so you know it’s not a UI issue.
    

### Common gotchas to check

1. **.env location** — it must be beside `app.py`. The code above points directly to that path.
    
2. **Warehouse** — make sure a SQL Warehouse exists and your token has permission. Test the same token in Databricks SQL UI.
    
3. **HTTP Path** — must be the **Warehouse** http path (starts with `/sql/1.0/warehouses/...`), not a cluster path.
    
4. **Catalog/Schema/View names** — match exactly what you used in your Databricks dashboard:
    
    - `v_position_overall`
        
    - `v_position_with_highest_pnl`
        
    - `v_sum_pnl_by_allotment`
        
    - `v_top_ten_instruments`
        
5. **Firewall / Proxy** — on corporate networks Git Bash sometimes uses different proxies. If connection fails only in Git Bash, try in **PowerShell** or set `HTTPS_PROXY` env var if your org requires it.
    

Once you see rows coming back, re-enable your charts below as before. If a query still doesn’t return, paste the error message that appears in the Streamlit page and I’ll pinpoint it.