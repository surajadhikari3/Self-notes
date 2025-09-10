You got it—here’s the **minimal, copy-paste recipe** to read **Kafka → Deephaven (local)** and render your **real-time dashboard**. No Docker needed.

---

# 0) One-time setup (local)

**Deephaven (no Docker)**

```bash
python -m venv dh-env
# Win: dh-env\Scripts\activate    Mac/Linux: source dh-env/bin/activate
python -m pip install --upgrade pip
python -m pip install deephaven-server
```

Create `start_dh.py`:

```python
from deephaven_server import Server
s = Server(port=10000, jvm_args=["-Xmx4g"])
s.start()
print("Deephaven IDE → http://localhost:10000/ide/  (Ctrl+C to stop)")
import time;  [time.sleep(60) for _ in iter(int,1)]
```

Run: `python start_dh.py` and open **[http://localhost:10000/ide/](http://localhost:10000/ide/)**.

**Kafka (local KRaft)**

- Install Kafka and start a broker on `localhost:9092`.
    
- Create topic:
    

```bash
kafka-topics --bootstrap-server localhost:9092 --create --topic positions --partitions 3 --replication-factor 1
```

(Optional) **Test producer** (quick JSON generator):

```bash
pip install kafka-python
python - << "PY"
import json, time, uuid, random, string
from datetime import datetime, timezone
from kafka import KafkaProducer
SOURCES=["GED","FIXED INCOME","COMMODITIES"]
ALLOT=["GovernmentBond","CorporateBond","Futures","Options","CommodityFund","Equity","REIT"]
producer=KafkaProducer(bootstrap_servers="localhost:9092",
                       value_serializer=lambda v: json.dumps(v).encode())
def rand(n,alphabet): return "".join(random.choices(alphabet,k=n))
while True:
    now=datetime.now(timezone.utc).isoformat()
    for _ in range(60):
        src=random.choice(SOURCES)
        msg={
          "_source_system":src,
          "allotment":random.choice(ALLOT),
          "instrumentCode":f"{src.split()[0][:3].upper()}{random.randint(1,99999):05d}",
          "positionId":str(uuid.uuid4()),
          "pnl":round(random.gauss(0,300),2),
          "mtm":round(random.uniform(100,50000),2),
          "isin":rand(12,string.ascii_uppercase+"0123456789"),
          "cusip":rand(9,string.ascii_uppercase+"0123456789"),
          "qty":random.randint(1,300),
          "price":round(random.uniform(5,500),2),
          "event_time":now
        }
        producer.send("positions", value=msg)
    producer.flush(); time.sleep(1)
PY
```

---

# 1) Deephaven: read Kafka and build the 4-widget dashboard

In the Deephaven IDE, create a **Python** script and paste all of this:

```python
# === Kafka → Deephaven live dashboard ===
from deephaven import kafka, dtypes as dht
from deephaven.agg import sum_
import deephaven.plot.express as dx
from deephaven.time import to_datetime

# ---- 1) Define JSON schema of your Kafka message (value) ----
value_spec = kafka.json_spec([
    ("_source_system", dht.string),
    ("allotment",      dht.string),
    ("instrumentCode", dht.string),
    ("positionId",     dht.string),
    ("pnl",            dht.double),
    ("mtm",            dht.double),
    ("isin",           dht.string),
    ("cusip",          dht.string),
    ("qty",            dht.int_),
    ("price",          dht.double),
    ("event_time",     dht.string),  # ISO-8601
])

# ---- 2) Consume the topic as a LIVE append table (push-based) ----
positions = kafka.consume(
    {'bootstrap.servers': 'localhost:9092', 'auto.offset.reset': 'latest'},
    topic='positions',
    value_spec=value_spec,
    table_type='append'
)

# Optional: parse event_time to an Instant (useful for windowing)
positions = positions.update(["EventTime = toDatetime(event_time)"])

# ---- 3) Parameter: pick your _source_system (GED / FIXED INCOME / COMMODITIES) ----
SOURCE = "GED"

# Slice the live table once; all widgets reuse this
pos = positions.where(f"_source_system == `{SOURCE}`")

# ---- 4) Widget 1: Position Data (overall table) ----
overall = pos.view([
    "_source_system", "allotment", "instrumentCode", "positionId",
    "pnl", "mtm", "isin", "cusip", "qty", "price", "EventTime"
])

# ---- 5) Widget 2: Highest P&L (single row) ----
highest = pos.sort_descending("pnl").head(1)

# ---- 6) Widget 3: Sum of P&L by allotment (bar) ----
pnl_by_allot = (
    pos.agg_by([sum_("total_pnl = pnl")], by=["allotment"])
       .sort_descending("total_pnl")
       .update(["sign = (total_pnl >= 0) ? `Gain` : `Loss`"])
)
bar_allot = dx.bar(
    pnl_by_allot, x="total_pnl", y="allotment", color="sign",
    title=f"Sum of P&L by Allotment • {SOURCE}"
)

# ---- 7) Widget 4: Top-10 instruments by P&L (bar) ----
top10 = (
    pos.agg_by([sum_("total_pnl = pnl")], by=["instrumentCode"])
       .sort_descending("total_pnl").head(10)
       .update(["sign = (total_pnl >= 0) ? `Gain` : `Loss`"])
)
bar_top10 = dx.bar(
    top10, x="total_pnl", y="instrumentCode", color="sign",
    title=f"Top 10 Instruments by P&L • {SOURCE}"
)

# ---- 8) (Optional) KPIs table ----
kpis = pos.agg_by([sum_("total_pnl = pnl")], by=[])
```

### What you’ll see

- `positions` ticks as Kafka messages arrive (no polling).
    
- `overall`, `highest`, `pnl_by_allot`/`bar_allot`, `top10`/`bar_top10` all **update in real time**.
    
- Change `SOURCE = "FIXED INCOME"` or `"COMMODITIES"`, run that line again to switch the entire dashboard.
    

> Tip: arrange your tables & charts in the UI and **File → Save Layout**.

---

## Quick checks if something doesn’t tick

- In the Deephaven console, run `positions.size()` — it should increase over time.
    
- Ensure your producer is writing to **topic `positions`** on **localhost:9092**.
    
- If you changed message fields, update `value_spec` accordingly.
    
- If you see offset errors, stop/start the script (keeps consuming from `latest` by default).
    

---

That’s it—you’re reading Kafka in Deephaven locally and rendering the same four widgets, with **true push-based updates**.


---------------------------------


You’re in **Git Bash / MINGW64** and your venv isn’t actually being used. Don’t try to “activate” it (your bash doesn’t have the usual helpers). Just call the **venv’s python.exe directly** for everything—creating, installing, and running. That avoids PowerShell/CMD entirely.

Here’s the exact sequence to fix the “No module named deephaven(_server)” error and start Deephaven from **Git Bash**:

---

## 1) Create the venv (using the system Python once)

> If your Python is elsewhere, adjust the first path accordingly.

```bash
/c/Users/TAP3507/AppData/Local/Programs/Python/Python310/python.exe \
  -m venv /c/Users/TAP3507/source/test_folder/dh-dashboard/dh-env
```

## 2) Install packages **into the venv** (no activation)

```bash
/c/Users/TAP3507/source/test_folder/dh-dashboard/dh-env/Scripts/python.exe \
  -m pip install --upgrade pip

/c/Users/TAP3507/source/test_folder/dh-dashboard/dh-env/Scripts/python.exe \
  -m pip install deephaven-server
```

> If you previously saw “Defaulting to user installation…”, this will fix it—the install now goes into `dh-env\Lib\site-packages`.

## 3) Verify the install

```bash
/c/Users/TAP3507/source/test_folder/dh-dashboard/dh-env/Scripts/python.exe \
  -c "import sys, deephaven_server; print('OK', deephaven_server.__version__, sys.executable)"
```

You should see `OK <version> C:\Users\TAP3507\...\dh-env\Scripts\python.exe`.

## 4) Start Deephaven (no activation needed)

Create `start_dh.py` in `C:\Users\TAP3507\source\test_folder\dh-dashboard\`:

```python
from deephaven_server import Server
s = Server(port=10000, jvm_args=["-Xmx2g"])  # needs JDK 17+
s.start()
print("Deephaven IDE → http://localhost:10000/ide/  (Ctrl+C to stop)")
import time;  time.sleep(999999)
```

Run it with the venv’s Python:

```bash
/c/Users/TAP3507/source/test_folder/dh-dashboard/dh-env/Scripts/python.exe \
  start_dh.py
```

Open **[http://localhost:10000/ide/](http://localhost:10000/ide/)**.

---

### Why your earlier attempts failed

- You ran `pip install` without the venv active → it installed to **user site** (`AppData\Roaming\Python\Python310\site-packages`) not your venv.
    
- Git Bash on your machine doesn’t have `sed`/`which`, so `source dh-env/Scripts/activate` breaks. We avoid activation entirely by calling the venv’s **full path** `python.exe`.
    
- Using `python test.py` in Git Bash ran the **system** Python (no DH module), not the venv.
    

---

### Quick sanity checks (optional)

```bash
# Show where that Python will import from (should be ...\dh-env\Lib\site-packages)
"/c/Users/TAP3507/source/test_folder/dh-dashboard/dh-env/Scripts/python.exe" -c "import sys; print(sys.path[0]); print(sys.executable)"

# Show that deephaven-server is in that interpreter's site-packages
"/c/Users/TAP3507/source/test_folder/dh-dashboard/dh-env/Scripts/python.exe" -m pip show deephaven-server
```

> If you get any Java errors starting the server, make sure **JDK 17+** is installed and `JAVA_HOME` points to it. You can run (in Git Bash):  
> `"/c/Program Files/Java/jdk-17/bin/java.exe" -version`

---

If you want, paste back the output of step 3 (the `OK ...` line). If it still says “ModuleNotFoundError”, I’ll adjust the paths to match your exact directory layout


-----------------------------


Thanks for the screenshot—that stack trace is super helpful. Two things are happening:

1. The server is trying to create/use a **local working folder** called `./.dh_py`, but your current shell/cwd combo (Git Bash on Windows) is confusing the path handling → `OSError: [Errno 2] No such file or directory: './.dh_py'`.
    
2. It also tried to use a **layouts** folder that ended up as `'/layouts'` (a Unix-root path that doesn’t exist on Windows) → “Invalid path: `/layouts`”.
    

Both are easy to fix by 1) running from a simple Windows path you own and 2) telling Deephaven explicitly where to keep its workspace and layouts using absolute Windows paths.

## Do this (Git Bash friendly; no activation needed)

1. **Pick a clean local folder** (no network drive). I’ll use:
    

```
C:\Users\TAP3507\dh-local
```

2. **Create the folders Deephaven will use:**
    

```bash
mkdir -p /c/Users/TAP3507/dh-local/workspace
mkdir -p /c/Users/TAP3507/dh-local/layouts
```

3. **Create a fresh start script** `start_dh.py` in `C:\Users\TAP3507\dh-local\`:
    

```python
# start_dh.py
import os
from deephaven_server import Server

# Use explicit absolute Windows paths (no leading slashes)
WORKSPACE = r"C:\Users\TAP3507\dh-local\workspace"
LAYOUTS   = r"C:\Users\TAP3507\dh-local\layouts"

# make sure they exist
os.makedirs(WORKSPACE, exist_ok=True)
os.makedirs(LAYOUTS,   exist_ok=True)

# Tell Deephaven where to put notebooks/workspace & layouts
jvm_args = [
    "-Xmx2g",
    f"-Ddeephaven.workspace={WORKSPACE}",
    f"-Ddeephaven.layouts.dir={LAYOUTS}",
]

s = Server(port=10000, jvm_args=jvm_args)
s.start()
print("Deephaven IDE → http://localhost:10000/ide/  (Ctrl+C to stop)")

# keep process alive
import time;  time.sleep(10**9)
```

> Those system properties force Deephaven to use **Windows absolute paths**, avoiding `./.dh_py` and `/layouts`.

4. **Run it using your venv’s Python (no activation)**
    

```bash
/c/Users/TAP3507/source/test_folder/dh-dashboard/dh-env/Scripts/python.exe \
  /c/Users/TAP3507/dh-local/start_dh.py
```

If Java 17+ is installed, you should now see it come up cleanly. Open **[http://localhost:10000/ide/](http://localhost:10000/ide/)**.

---

### If you still see a path error

- Make sure you’re running **from a local folder** (not a network share).
    
- Double-check the two paths in the script are valid on your machine.
    
- If you changed the username or base path, mirror it in both the `mkdir` commands and `WORKSPACE`/`LAYOUTS` strings.
    

---

### Why this works

- Git Bash sometimes converts or normalizes relative paths oddly on Windows. By giving Deephaven **absolute Windows paths** via JVM args, we bypass that ambiguity.
    
- Creating the directories up front removes the “No such file or directory: './.dh_py'” failure mode.
    
- Setting `-Ddeephaven.layouts.dir` prevents it from defaulting to a bogus root path like `'/layouts'`.
    

---

If you want, paste the very first 10–15 lines of output after you run the command above. If there’s still an error, I’ll diagnose it from the exact message and tweak the property names/paths accordingly.


----------------------


Thanks for the screenshot—this tells us exactly what’s going on.

### What the error means

Deephaven is trying to use a **layouts directory at `/layouts`** and it can’t find it:

```
Invalid ARGUMENT: Invalid path: /layouts
...
FilesystemStorageServiceGrpcImpl.Factory
```

On Windows, a path like `/layouts` resolves to `C:\layouts`. Since that folder doesn’t exist, the server fails during startup.

### Fastest fix (2 commands)

Create the two folders Deephaven expects and start again:

```bash
# in Git Bash
mkdir -p /c/layouts
mkdir -p /c/Users/TAP3507/.dh_py
```

Then run your start script again with your venv’s Python:

```bash
/c/Users/TAP3507/source/test_folder/dh-dashboard/dh-env/Scripts/python.exe \
  /c/Users/TAP3507/dh-local/start_dh.py
```

Open: [http://localhost:10000/ide/](http://localhost:10000/ide/)

---

## Why this happens

- Deephaven loads default config from  
    `C:\Users\TAP3507\AppData\Roaming\Deephaven Data Labs\deephaven\config`  
    and in that config, the **layouts path** defaults to `/layouts`.
    
- Because that folder doesn’t exist, the server aborts.
    
- Creating `C:\layouts` (i.e., `/layouts` from Git Bash) satisfies the config and the server will start.
    

---

## Optional: avoid using `C:\layouts`

If you prefer to keep everything under your user directory, create your own start script that **overrides** the default paths (this avoids `/layouts` entirely):

```python
# start_dh.py  (place in C:\Users\TAP3507\dh-local)
import os
from deephaven_server import Server

WORKSPACE = r"C:\Users\TAP3507\dh-local\workspace"
LAYOUTS   = r"C:\Users\TAP3507\dh-local\layouts"

os.makedirs(WORKSPACE, exist_ok=True)
os.makedirs(LAYOUTS,   exist_ok=True)

jvm_args = [
    "-Xmx2g",
    f"-Ddeephaven.workspace={WORKSPACE}",
    f"-Ddeephaven.layouts.dir={LAYOUTS}",
]

s = Server(port=10000, jvm_args=jvm_args)
s.start()
print("Deephaven IDE → http://localhost:10000/ide/  (Ctrl+C to stop)")
import time; time.sleep(10**9)
```

Run it:

```bash
/c/Users/TAP3507/source/test_folder/dh-dashboard/dh-env/Scripts/python.exe \
  /c/Users/TAP3507/dh-local/start_dh.py
```

> If this still falls back to `/layouts`, it means your local default config is overriding the properties. The quick workaround is the **first method** (just create `C:\layouts`). We can later adjust the config file in `AppData\Roaming\Deephaven Data Labs\deephaven\config` to point layouts elsewhere.

---

## If anything still fails

Send me the **first 10–15 lines** after you start the script (just the header + the first exception line). But in most cases, creating `C:\layouts` (and `C:\Users\TAP3507\.dh_py`) makes the server boot immediately.

---------------------------


Perfect—that explains it. Since there’s **no config** under:

```
C:\Users\TAP3507\AppData\Roaming\Deephaven Data Labs\deephaven\config
```

the server falls back to a bad default (`/layouts`). Let’s **create the config and point Deephaven to your own folders** so it stops looking for `/layouts`.

---

## Do this exactly (Git Bash friendly)

### 1) Create folders Deephaven will use

```bash
mkdir -p /c/Users/TAP3507/dh-local/workspace
mkdir -p /c/Users/TAP3507/dh-local/layouts
```

### 2) Create the Deephaven config folder

```bash
mkdir -p "/c/Users/TAP3507/AppData/Roaming/Deephaven Data Labs/deephaven/config"
```

### 3) Create a **deephaven.properties** file with explicit Windows paths

```bash
cat > "/c/Users/TAP3507/AppData/Roaming/Deephaven Data Labs/deephaven/config/deephaven.properties" << 'EOF'
# Force Deephaven to use these directories (avoid /layouts default)
deephaven.layouts.dir = C:\\Users\\TAP3507\\dh-local\\layouts
deephaven.workspace   = C:\\Users\\TAP3507\\dh-local\\workspace

# Extra aliases some builds look at (harmless if unused)
deephaven.console.layouts.dir = C:\\Users\\TAP3507\\dh-local\\layouts
deephaven.ide.layouts.dir     = C:\\Users\\TAP3507\\dh-local\\layouts
deephaven.server.notebook.dir = C:\\Users\\TAP3507\\dh-local\\workspace
deephaven.server.notebook.filesystem.root = C:\\Users\\TAP3507\\dh-local\\workspace
EOF
```

> Note the **double backslashes** (`\\`) — required in Java properties for Windows paths.

### 4) Start the server (no activation; use your venv’s python)

```bash
"/c/Users/TAP3507/source/test_folder/dh-dashboard/dh-env/Scripts/python.exe" \
  "/c/Users/TAP3507/dh-local/start_dh.py"
```

Open: **[http://localhost:10000/ide/](http://localhost:10000/ide/)**

---

## If you still see `/layouts` in the error

That would mean the properties weren’t loaded. Quick checks:

```bash
# confirm the file exists and contents look right
ls -l "/c/Users/TAP3507/AppData/Roaming/Deephaven Data Labs/deephaven/config/deephaven.properties"
cat "/c/Users/TAP3507/AppData/Roaming/Deephaven Data Labs/deephaven/config/deephaven.properties"
```

Then try the **belt-and-suspenders** approach: also pass the same values via JVM args in your `start_dh.py`:

```python
# start_dh.py (replace with this if needed)
import os
from deephaven_server import Server

BASE = r"C:\Users\TAP3507\dh-local"
WORKSPACE = rf"{BASE}\workspace"
LAYOUTS   = rf"{BASE}\layouts"
os.makedirs(WORKSPACE, exist_ok=True)
os.makedirs(LAYOUTS,   exist_ok=True)

jvm_args = [
    "-Xmx2g",
    f"-Ddeephaven.layouts.dir={LAYOUTS}",
    f"-Ddeephaven.workspace={WORKSPACE}",
    f"-Ddeephaven.console.layouts.dir={LAYOUTS}",
    f"-Ddeephaven.ide.layouts.dir={LAYOUTS}",
    f"-Ddeephaven.server.notebook.dir={WORKSPACE}",
    f"-Ddeephaven.server.notebook.filesystem.root={WORKSPACE}",
]
s = Server(port=10000, jvm_args=jvm_args)
s.start()
print("Deephaven IDE → http://localhost:10000/ide/  (Ctrl+C to stop)")
import time; time.sleep(10**9)
```

Run again with the same command in step 4.

---

## One more safety net (works instantly)

If for any reason the config still doesn’t register, **create the folder it’s asking for**:

```bash
mkdir -p /c/layouts
```

That’s the path Deephaven complains about (`/layouts` → `C:\layouts`). Creating it guarantees startup, even without the properties file.

---

Once the IDE is up, we can drop in the **Kafka consumer** code and you’ll have your push-based dashboard running.

------------------------------------


Thanks for sticking with it — your screenshots still show Deephaven boot failing with:

```
Invalid ARGUMENT: Invalid path: /layouts
```

That means Java is still reading a config that points to **`/layouts`** and **ignoring** the file you created under AppData. Let’s force Deephaven to use _our_ config and paths, so it can’t fall back.

Below are copy-paste steps for **Git Bash** only (no PowerShell).

---

## 1) Make clean folders we control

```bash
# workspace & layouts (under your user)
mkdir -p /c/Users/TAP3507/dh-local/workspace
mkdir -p /c/Users/TAP3507/dh-local/layouts

# a dedicated config dir (separate from AppData)
mkdir -p /c/Users/TAP3507/dh-local/config
```

## 2) Write an explicit **deephaven.properties** in that config dir

```bash
cat > "/c/Users/TAP3507/dh-local/config/deephaven.properties" << 'EOF'
# Force Deephaven to use these dirs (Windows paths with double backslashes)
deephaven.layouts.dir = C:\\Users\\TAP3507\\dh-local\\layouts
deephaven.workspace   = C:\\Users\\TAP3507\\dh-local\\workspace

# Extra aliases (harmless if not used)
deephaven.console.layouts.dir = C:\\Users\\TAP3507\\dh-local\\layouts
deephaven.ide.layouts.dir     = C:\\Users\\TAP3507\\dh-local\\layouts
deephaven.server.notebook.dir = C:\\Users\\TAP3507\\dh-local\\workspace
deephaven.server.notebook.filesystem.root = C:\\Users\\TAP3507\\dh-local\\workspace
EOF
```

Verify:

```bash
cat "/c/Users/TAP3507/dh-local/config/deephaven.properties"
```

## 3) Replace your `start_dh.py` with hard overrides

> We will 1) point Deephaven **explicitly** at our config directory, and 2) **also** pass the same directories via JVM args. This double-ensures no fallback to `/layouts`.

Create `C:\Users\TAP3507\dh-local\start_dh.py`:

```python
import os
from deephaven_server import Server

BASE = r"C:\Users\TAP3507\dh-local"
WORKSPACE = rf"{BASE}\workspace"
LAYOUTS   = rf"{BASE}\layouts"
CFG       = rf"{BASE}\config"

os.makedirs(WORKSPACE, exist_ok=True)
os.makedirs(LAYOUTS,   exist_ok=True)
os.makedirs(CFG,       exist_ok=True)

jvm_args = [
    "-Xmx2g",

    # 1) Tell DH which config directory to read (ours)
    f"-Ddeephaven.config.dir={CFG}",

    # 2) Also override the specific properties directly
    f"-Ddeephaven.layouts.dir={LAYOUTS}",
    f"-Ddeephaven.workspace={WORKSPACE}",
    f"-Ddeephaven.console.layouts.dir={LAYOUTS}",
    f"-Ddeephaven.ide.layouts.dir={LAYOUTS}",
    f"-Ddeephaven.server.notebook.dir={WORKSPACE}",
    f"-Ddeephaven.server.notebook.filesystem.root={WORKSPACE}",

    # 3) Debug: print loaded config on startup
    "-Ddeephaven.config.print=true",
]

s = Server(port=10000, jvm_args=jvm_args)
s.start()
print("Deephaven IDE → http://localhost:10000/ide/  (Ctrl+C to stop)")

import time; time.sleep(10**9)
```

## 4) Start using your venv’s Python (no activation)

```bash
"/c/Users/TAP3507/source/test_folder/dh-dashboard/dh-env/Scripts/python.exe" \
  "/c/Users/TAP3507/dh-local/start_dh.py"
```

On startup you should now see lines printing the properties file it loaded and the values. The server should come up at **[http://localhost:10000/ide/](http://localhost:10000/ide/)**.

---

## 5) Guaranteed fallback (if anything still points at `/layouts`)

Create the exact folder it’s complaining about:

```bash
mkdir -p /c/layouts
```

That’s `C:\layouts` in Windows terms and will satisfy any leftover default. You can keep both: your custom dirs **and** `C:\layouts` as a safety net.

---

## Why this works

- `-Ddeephaven.config.dir=<dir>` tells Deephaven to read config **only** from that directory.
    
- Passing the **same paths** as JVM properties makes them win even if some other config gets picked up.
    
- `config.print=true` helps confirm the active values at runtime.
    

---

If you still hit an error, please grab just the **first 10–15 lines** of startup (where it prints the config values) and the **first line** of the exception (the “Invalid path: …” line). With that, I can pinpoint the remaining override.


------------------------------------

jj

I can see clearly in your screenshot what’s happening now:

👉 The error at the bottom says:

```
RuntimeError: The Deephaven Enterprise/Community Python Package (deephaven-core on pypi)
is installed for the current Python environment. If continuing issues, please uninstall 
the deephaven packages and re-install.
```

That means `deephaven-core` **did install**, but when you run `python -m deephaven_server` it’s not finding a proper embedded server because of **conflicting or partial installs**.

---

## 🔑 Key points

- You have both `deephaven-core` and `deephaven-plugin` installed.
    
- The package that provides the actual **embedded server** entrypoint is **`deephaven-server`**.
    
- If you only install `deephaven-core`, you’ll get client + API but not the launcher.
    

---

## ✅ Clean fix steps (bash)

Let’s nuke the broken venv and reinstall with the **server package included**.

```bash
# 1) remove old venv
deactivate 2>/dev/null || true
rm -rf dh-venv

# 2) create fresh venv
python3 -m venv dh-venv
source dh-venv/bin/activate

# 3) upgrade local pip tools
python -m pip install --upgrade pip setuptools wheel

# 4) install Deephaven pinned versions (server + core + plugin)
pip install "deephaven-core==0.39.8" "deephaven-server==0.39.8" "deephaven-plugin==0.6.9" "pyarrow==14.0.2" "jpy==1.1.0" "numpy==1.26.4" "pandas==2.2.2"
```

---

## 🚀 Starting the server

Option A: Use the CLI

```bash
deephaven server --port 10000
```

Option B: Use Python API

```bash
python - <<'PY'
from deephaven_server import Server
s = Server(port=10000)
s.start()
print("Deephaven started at http://localhost:10000")
s.await_shutdown()
PY
```

Now open [http://localhost:10000](http://localhost:10000/).

---

## 📝 Notes

- Make sure Java 11 or 17 is installed, and `JAVA_HOME` points to it:
    
    ```bash
    export JAVA_HOME="/c/Program Files/Java/jdk-17"
    export PATH="$JAVA_HOME/bin:$PATH"
    ```
    
- If you still see “deephaven_server not found”, check:
    
    ```bash
    pip show deephaven-server
    ```
    
    If missing, the install failed (try again inside the venv).
    

---

checkk


Thanks—this is the same **“/layouts”** problem on Windows. The fix is to point Deephaven to a **real folder** for layouts and start the server with that path.

Below are two reliable ways. Pick one.

---

## A) Start via Python (works everywhere)

```bash
# 1) choose a Windows-native folder for Deephaven data & layouts
mkdir -p /c/dhdata/layouts

# 2) activate your venv (adjust path)
source dh-venv/bin/activate

# 3) launch the embedded server with an explicit layout dir
python - <<'PY'
import os
from deephaven_server import Server

data_dir   = r"C:\dhdata"
layout_dir = r"C:\dhdata\layouts"

os.makedirs(layout_dir, exist_ok=True)

# Important: give JVM Windows paths with forward slashes
jvm_args = [
    f"-Ddeephaven.console.layout.dir={layout_dir.replace('\\','/')}",
    f"-Ddeephaven.data.dir={data_dir.replace('\\','/')}",
    f"-Ddeephaven.cache.dir={data_dir.replace('\\','/')}/cache",
    f"-Ddeephaven.config.dir={data_dir.replace('\\','/')}/config",
]

s = Server(port=10000, jvm_args=jvm_args)
s.start()
print("Deephaven running at http://localhost:10000")
s.await_shutdown()
PY
```

**Why this works:** the JVM was trying to use an invalid path `"/layouts"`; we override it with a valid Windows path (e.g., `C:/dhdata/layouts`) using system properties.

---

## B) Start via CLI with JVM args

```bash
# create folders
mkdir -p /c/dhdata/layouts /c/dhdata/cache /c/dhdata/config

# activate venv
source dh-venv/bin/activate

# launch with explicit dirs
deephaven server \
  --jvm-arg "-Ddeephaven.console.layout.dir=C:/dhdata/layouts" \
  --jvm-arg "-Ddeephaven.data.dir=C:/dhdata" \
  --jvm-arg "-Ddeephaven.cache.dir=C:/dhdata/cache" \
  --jvm-arg "-Ddeephaven.config.dir=C:/dhdata/config" \
  --port 10000
```

---

## If you still see the error

- Make sure **Java** is available in this shell:
    
    ```bash
    export JAVA_HOME="/c/Program Files/Java/jdk-17"
    export PATH="$JAVA_HOME/bin:$PATH"
    java -version
    ```
    
- Confirm the **paths exist** and are writable (`/c/dhdata/layouts` etc.).
    
- Prefer **forward slashes** in JVM args (`C:/dhdata/...`).
    
- If running in **Git Bash**, you can also set env vars before starting:
    
    ```bash
    export DEEPHAVEN_DATA_DIR="C:/dhdata"
    export DEEPHAVEN_CACHE_DIR="C:/dhdata/cache"
    export DEEPHAVEN_CONFIG_DIR="C:/dhdata/config"
    ```
    
    (and still pass `-Ddeephaven.console.layout.dir=...` to be explicit).
    

---

### Quick fallback (no Windows path quirks)

If you just need it up quickly, run the Docker image:

```bash
docker run --rm --name deephaven -p 10000:10000 ghcr.io/deephaven/server:latest
```

---

If you paste the **first 10 lines** of the stack trace after trying A or B, I’ll tune the exact property names/paths for your setup.