# SQL Explorer Tab — Technical Documentation

**Project:** `pm5_combined_dashboard.py`  
**Tab Label:** `🗄️ SQL Explorer` (Tab 4)  
**Function:** `tab4_sql_explorer()`  
**Source File:** `pm5_combined_dashboard.py` — Lines 1883–2756  

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture and Dependencies](#architecture-and-dependencies)
3. [Database Connection](#database-connection)
4. [Read-Only Safety Guard](#read-only-safety-guard)
5. [Query Engine — `run_query()`](#query-engine----run_query)
6. [Database and Table Constants](#database-and-table-constants)
7. [Tab 4 UI Layout and Sections](#tab-4-ui-layout-and-sections)
   - 7.1 [Quick Query Buttons](#71-quick-query-buttons)
   - 7.2 [SQL Query Runner](#72-sql-query-runner)
   - 7.3 [Available Databases Browser](#73-available-databases-browser)
   - 7.4 [Browse Tables](#74-browse-tables)
   - 7.5 [Browse Columns](#75-browse-columns)
8. [Batch Profiled GSM Analyzer](#batch-profiled-gsm-analyzer)
   - 8.1 [Purpose](#81-purpose)
   - 8.2 [Checkpoint System](#82-checkpoint-system)
   - 8.3 [Date Range Input](#83-date-range-input)
   - 8.4 [Track Fetching](#84-track-fetching)
   - 8.5 [Per-Track Processing — `_process_single_track_batch()`](#85-per-track-processing----_process_single_track_batch)
   - 8.6 [Width Position Calculation](#86-width-position-calculation)
   - 8.7 [Time Determination Priority Chain](#87-time-determination-priority-chain)
   - 8.8 [Per-Set Time Narrowing](#88-per-set-time-narrowing)
   - 8.9 [GSM Calculation — PI 600-Point Profile](#89-gsm-calculation----pi-600-point-profile)
   - 8.10 [ULMA Substance GSM](#810-ulma-substance-gsm)
   - 8.11 [Result Status Codes](#811-result-status-codes)
   - 8.12 [Progress Display and Summary Metrics](#812-progress-display-and-summary-metrics)
   - 8.13 [Excel Download](#813-excel-download)
9. [Key Database Tables Reference](#key-database-tables-reference)
10. [Data Flow Diagram](#data-flow-diagram)
11. [Helper Functions Reference](#helper-functions-reference)
12. [Error Handling and Edge Cases](#error-handling-and-edge-cases)
13. [Usage Guide](#usage-guide)

---

## Overview

The **SQL Explorer** is Tab 4 of the `pm5_combined_dashboard.py` Streamlit application. It provides **two major capabilities**:

1. **Interactive SQL Workspace** — A self-service SQL environment for operators and engineers to directly query the production SQL Server databases. It includes one-click query shortcuts, a free-form SQL editor, and a built-in database/table/column browser.

2. **Batch Profiled GSM Analyzer** — A high-performance batch processing engine that fetches every wound reel in a user-defined date range and calculates the full **Profiled GSM** (grams per square metre) for each reel using PI Server quality profile data. Results are crash-safe (checkpoint to disk) and downloadable as Excel.

The SQL Explorer is designed so that **any user**, even without SQL knowledge, can explore production data safely — the system enforces read-only access at the code level, blocking all write/destructive SQL statements.

---

## Architecture and Dependencies

```
pm5_combined_dashboard.py
│
├── Database Layer (pyodbc)
│   ├── init_connection()          — singleton connection, cached with @st.cache_resource
│   ├── _is_read_only(query)       — security guard, blocks DML/DDL
│   └── run_query(query)           — cached execution, returns (columns, rows)
│
├── Tab 4 UI (tab4_sql_explorer)
│   ├── Quick Query Buttons        — 3 pre-built SELECT queries
│   ├── SQL Query Runner           — free-form text_area + "Run Query" button
│   ├── Available Databases        — queries sys.databases
│   ├── Browse Tables              — queries INFORMATION_SCHEMA.TABLES
│   └── Browse Columns             — queries INFORMATION_SCHEMA.COLUMNS
│
└── Batch Profiled GSM Analyzer (nested inside Tab 4)
    ├── Checkpoint System          — JSON file on disk, crash-recovery
    ├── _process_single_track_batch() — full Tab 2 logic per track
    ├── PI Server Integration      — 600-point cross-machine profile
    └── Excel Export               — openpyxl, two-sheet workbook
```

**Key Python packages used:**

| Package | Purpose |
|---|---|
| `streamlit` | UI framework (widgets, layout, dataframes) |
| `pyodbc` | SQL Server ODBC connectivity |
| `pandas` | DataFrame manipulation, datetime parsing |
| `numpy` | Zone-averaging math for ULMA GSM |
| `openpyxl` | Excel generation for batch download |
| `json` | Checkpoint file serialization |
| `os` | File path management for checkpoints |
| `datetime.timedelta` | Time arithmetic for ULMA/WOUND fallbacks |

---

## Database Connection

**Function:** `init_connection()` — lines 60–79

```python
@st.cache_resource
def init_connection():
    connection_file = r"F:\Env_new\pm5 Run chart\connection.txt"
    if not os.path.exists(connection_file):
        connection_file = "connection.txt"
    with open(connection_file, "r") as f:
        connection_string = f.read().strip()
    return pyodbc.connect(connection_string)
```

**How it works:**

- The `@st.cache_resource` decorator ensures the connection is created **once per app session** and reused across all queries — equivalent to a singleton connection pool.
- The connection string is read from a text file (`connection.txt`) stored on the server, keeping credentials out of source code.
- The primary path is `F:\Env_new\pm5 Run chart\connection.txt` (production Windows server). If not found, it falls back to `connection.txt` in the script's own directory.
- On failure, `conn` is set to `None` and all database-dependent sections display a "Database not connected" warning.

**Global variable:**  
`conn = init_connection()` is called at module level and stored in `conn`. The `run_query()` function uses this `conn` object.

---

## Read-Only Safety Guard

**Constants and function — lines 82–94**

```python
BLOCKED_KEYWORDS = [
    "INSERT", "UPDATE", "DELETE", "DROP", "ALTER",
    "CREATE", "TRUNCATE", "EXEC", "EXECUTE",
    "MERGE", "GRANT", "REVOKE", "DENY"
]

def _is_read_only(query: str) -> bool:
    q = query.strip().upper()
    for kw in BLOCKED_KEYWORDS:
        if q.startswith(kw) or f" {kw} " in q:
            return False
    return True
```

**Design choices:**

- Checks both `startswith` (catches statements that begin with a blocked word) and `" {kw} "` (catches inline occurrences like `DROP TABLE` inside a longer string).
- The check is case-insensitive (`.upper()` before comparison).
- Any query that fails this check is **rejected before being sent to the database** — it never reaches pyodbc. The user sees an `st.error()` message.
- This is a defense-in-depth measure. The database user account should also be configured with read-only permissions, but this guard provides an additional software-level barrier.

---

## Query Engine — `run_query()`

**Lines 97–120**

```python
@st.cache_data(ttl=120)
def run_query(query: str):
    if not _is_read_only(query):
        st.error("🚫 Query blocked: only SELECT statements are allowed.")
        return [], []
    try:
        cursor = conn.cursor()
        cursor.execute(query)
        columns = [col[0].upper() for col in cursor.description]
        rows = cursor.fetchall()
        return columns, rows
    except Exception as e:
        st.error(f"Query error: {e}")
        return [], []
```

**Key behaviors:**

| Behavior | Detail |
|---|---|
| **Read-only check** | Calls `_is_read_only()` before any execution |
| **Result caching** | `@st.cache_data(ttl=120)` — identical queries return cached results for 2 minutes |
| **Column normalization** | All column names are converted to UPPERCASE for consistent access |
| **Return type** | Always returns `(list[str], list[tuple])` — columns and rows. Returns `([], [])` on any error |
| **Error display** | Calls `st.error()` in-place so the error appears in the UI where the query was triggered |

---

## Database and Table Constants

**Lines 35–38**

```python
WINDER_LOG_TABLE   = "[PSPDQUALDHI].[dbo].[invent_dhi_wound]"
ULMA_TIME_TABLE    = "[PSPDQUALDHI].[dbo].[OPTI_DISQUA_11_2_ULMA_TIME]"
ULMA_REMARKS_TABLE = "[PSPDQUALDHI].[dbo].[jumbo_ulma_remarks]"
DISPATCH_TABLE     = "[PSPDQUALDHI].[dbo].[disprd26_dhi]"
```

These constants are used throughout the Batch GSM Analyzer to build parameterized SQL queries. They use fully-qualified three-part SQL Server object names (`database.schema.table`) to avoid ambiguity.

---

## Tab 4 UI Layout and Sections

### 7.1 Quick Query Buttons

Three one-click buttons appear at the top of the tab. Each button populates the SQL results area immediately when clicked without requiring the user to type anything.

**Button 1 — Recent Reels (INVENT_DHI)**
```sql
SELECT TOP 10
    track_num, parent_id, width_ord, length_lineal,
    bsw_nominal, prod_mach_id, WOUND_TIME
FROM [PSPDQUALDHI].[dbo].[INVENT_DHI]
ORDER BY WOUND_TIME DESC
```
Shows the 10 most recently wound reels from the production inventory table.

**Button 2 — QCS Data Sample**
```sql
SELECT TOP 10
    JUMBO_ID, MACHINE_ID, QPARAM, TS_DT
FROM [eCubix_PI_Datalogger].[dbo].[OPTI_Data_Oracle_Log]
ORDER BY TS_DT DESC
```
Shows recent records from the QCS (Quality Control System) data logger, including quality parameter names (`QPARAM`) and their timestamps.

**Button 3 — ULMA Times Sample**
```sql
SELECT TOP 10
    JUMBO_ID, ULMATIME, MACHINE_ID
FROM [PSPDQUALDHI].[dbo].[OPTI_DISQUA_11_2_ULMA_TIME]
ORDER BY ULMATIME DESC
```
Shows the most recent ULMA (unwind/rewind) operation times, used as the primary time reference for GSM calculation.

---

### 7.2 SQL Query Runner

A free-form SQL text area that accepts any SELECT statement. After typing a query, the user clicks **"Run Query"** to execute it.

- Uses `st.text_area()` — multi-line input, suitable for complex queries.
- On button click, calls `run_query(query_text)`.
- Results are rendered with `st.dataframe()` — sortable, scrollable, full-width.
- All write operations are silently blocked by the read-only guard.

---

### 7.3 Available Databases Browser

Automatically queries the SQL Server instance for all accessible databases:

```sql
SELECT name FROM sys.databases ORDER BY name
```

- Results are displayed as a list/dataframe.
- This lets users discover what databases are available before writing queries.
- The selectbox populated here feeds into the Browse Tables section.

---

### 7.4 Browse Tables

User picks a database from a selectbox and clicks **"Show Tables"**. The app queries:

```sql
SELECT TABLE_SCHEMA, TABLE_NAME, TABLE_TYPE
FROM [<selected_database>].INFORMATION_SCHEMA.TABLES
ORDER BY TABLE_SCHEMA, TABLE_NAME
```

- `TABLE_TYPE` distinguishes `BASE TABLE` (actual tables) from `VIEW`.
- Results are rendered as a dataframe; clicking a row or entering a table name leads to the column browser.

---

### 7.5 Browse Columns

After selecting a table, clicking **"Show Columns"** queries:

```sql
SELECT
    COLUMN_NAME, DATA_TYPE,
    CHARACTER_MAXIMUM_LENGTH, IS_NULLABLE
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_NAME = '<selected_table>'
ORDER BY ORDINAL_POSITION
```

- `ORDINAL_POSITION` ordering ensures columns appear in the same order as in the actual table definition.
- `CHARACTER_MAXIMUM_LENGTH` shows max string length for `VARCHAR`/`NVARCHAR` fields, or `NULL` for numeric/datetime columns.
- This acts as an in-app schema viewer — users can understand a table's structure without needing SSMS or a DBA.

---

## Batch Profiled GSM Analyzer

### 8.1 Purpose

The Batch Profiled GSM Analyzer (nested inside Tab 4, lines ~2019–2756) is a **bulk computation engine**. Its purpose is:

> Given a start and end datetime, find all PM5 wound reels (TRACK_NUM) in that time window and calculate the **Profiled GSM** for each one using the same algorithm as the interactive Tab 2.

This is used for:
- End-of-shift or end-of-day GSM reporting
- Quality audits across a production run
- Feeding data to downstream Excel reports

**Requirements for the batch analyzer to work:**
1. SQL Server database must be connected (`conn is not None`)
2. OSIsoft PI Server must be connected (`PI_CONNECTED = True`) — GSM is calculated from PI profile data

---

### 8.2 Checkpoint System

Since processing hundreds of tracks can take minutes, a **crash-recovery checkpoint system** prevents work from being lost if the browser refreshes or the app restarts.

**Functions (lines ~2025–2056):**

```python
def _build_checkpoint_path(start_str, end_str):
    # Returns path like: batch_gsm_checkpoint_2024-07-01 06:00:00_to_2024-07-01 18:00:00.json
    safe_start = start_str.replace(":", "-").replace(" ", "_")
    safe_end   = end_str.replace(":", "-").replace(" ", "_")
    script_dir = os.path.dirname(os.path.abspath(__file__))
    fname = f"batch_gsm_checkpoint_{safe_start}_to_{safe_end}.json"
    return os.path.join(script_dir, fname)

def _load_checkpoint(path):
    if os.path.exists(path):
        with open(path, "r") as f:
            return json.load(f)
    return {}

def _save_checkpoint(path, results_dict):
    with open(path, "w") as f:
        json.dump(results_dict, f, default=str)
```

**Checkpoint file format:**
- A single JSON object where keys are `TRACK_NUM` strings and values are the full result dict for that track.
- Saved after **every single track** is processed.
- Unique per date range — changing the date range starts a fresh checkpoint file.

**Checkpoint UI controls:**
- `🚀 Analyze All Tracks in Date Range` — starts (or resumes) processing
- `🗑️ Clear Checkpoint` — deletes the checkpoint file for the current date range
- On load: displays `"📁 Found checkpoint with N previously processed tracks. New run will resume from where it left off."`

---

### 8.3 Date Range Input

Four inputs arranged in a 2×2 grid (date + time for start, date + time for end):

```
[ Start Date ] [ Start Time ]  [ End Date ] [ End Time ]
```

- `st.date_input` for dates (defaults to today)
- `st.time_input` for times (start defaults to `06:00`, end defaults to `18:00`)
- Combined into pandas Timestamps: `batch_start_dt = pd.Timestamp.combine(date, time)`

---

### 8.4 Track Fetching

When "Analyze All Tracks" is clicked, the app first fetches all matching tracks:

```sql
SELECT
    TRACK_NUM, PARENT_ID, SET_ID, CORE_ID,
    WIDTH_ACT, WIDTH_ORD, LENGTH_LINEAL, WOUND_TIME,
    BSW_NOMINAL, WGT_SCALED, BSW_ACT, PROD_MACH_ID, GRADE_SPEC
FROM [PSPDQUALDHI].[dbo].[invent_dhi_wound]
WHERE WOUND_TIME >= '<batch_start_dt>'
  AND WOUND_TIME <= '<batch_end_dt>'
  AND PROD_MACH_ID = '05'
  AND MILL_ID = 'BCM'
ORDER BY WOUND_TIME ASC
```

**Filters:**
- `PROD_MACH_ID = '05'` — only PM5 machine reels
- `MILL_ID = 'BCM'` — only BCM mill data

After fetching, tracks already in the checkpoint are skipped: only `[t for t in all_track_nums if t not in already_done]` are processed.

---

### 8.5 Per-Track Processing — `_process_single_track_batch()`

This inner function (lines 2078–2485) replicates the **complete Tab 2 Profiled GSM algorithm** for a single track number, in batch (non-interactive) mode.

**Input:** `track_num_str` — a string track number, e.g. `"123456"`

**Output:** A dict with 19 fields:

| Field | Type | Description |
|---|---|---|
| `TRACK_NUM` | str | Input track number |
| `PARENT_ID` | str | Jumbo roll ID |
| `SET_ID` | str | Set (cutting pattern) ID within the jumbo |
| `CORE_ID` | str | Core position identifier |
| `WOUND_TIME` | str | Timestamp when reel was wound |
| `WIDTH_ACT` | float | Actual reel width (mm) |
| `WGT_SCALED` | float | Scaled reel weight (kg) |
| `BSW_ACT` | float | Actual basis weight nominal |
| `PROFILED_GSM` | float | Position-weighted GSM from PI 600-pt profile |
| `OVERALL_GSM` | float | Machine-average GSM from PI data |
| `ULMA_PROFILED_GSM` | float | GSM from ULMA 12-zone substance data |
| `START_TIME` | str | Determined production start time |
| `END_TIME` | str | Determined production end time |
| `START_TIME_SOURCE` | str | How start time was derived |
| `END_TIME_SOURCE` | str | How end time was derived |
| `DECKLE_WIDTH` | float | Total machine cutting width (mm) |
| `START_WIDTH` | float | Reel's start position on deckle (mm from DS) |
| `END_WIDTH` | float | Reel's end position on deckle (mm from DS) |
| `PI_POINTS` | int | Number of active PI quality zones used |
| `SET_TIME_NARROWED` | bool | Whether time was narrowed to this specific set |
| `STATUS` | str | Final status code (see §8.11) |
| `ERROR` | str\|None | Error message if STATUS is not SUCCESS |

**Processing steps inside `_process_single_track_batch()`:**

**Step 1: Fetch track data**
- Queries `WINDER_LOG_TABLE` for `TRACK_NUM = '<track>'` with `PROD_MACH_ID = '05'` and `MILL_ID = 'BCM'`
- If not found → `STATUS = "NOT_FOUND"`
- Uses `iloc[0]` (earliest CORE_ID) if multiple records exist

**Step 2: Fetch all reels in the same set → width profile**
- Queries all records for `PARENT_ID + CORE_ID + PROD_MACH_ID`
- Applies `clean_ghost_reels_jumbowide()` to remove ghost/duplicate reels
- Computes numeric widths, track numbers, and set IDs for position calculation

**Step 3: Time determination** (see §8.7)

**Step 4: GSM Calculation** (see §8.9)

**Step 5: ULMA Substance GSM** (see §8.10)

---

### 8.6 Width Position Calculation

Each reel occupies a specific cross-machine position on the paper machine's deckle (the cutting pattern). The position must be known to extract the correct slice from the PI width profile.

**Standard case (multiple reels in set):**
```python
deckle_width = df_group['WIDTH_ORD_NUM'].sum()       # sum of all reel widths in this cutting pattern
tracks_after = df_group[df_group['TRACK_NUM_NUM'] > track_num_numeric]
winder_start = tracks_after['WIDTH_ORD_NUM'].sum()    # mm from Drive Side
winder_end   = winder_start + target_track_width
```

**Single-reel case:**  
When only one reel exists in the set (e.g. a full-width reel), the app queries `POS_ID_CUT` from the full jumbo to reconstruct the deckle layout:
```python
my_pos_cut    = pd.to_numeric(track_data.get('POS_ID_CUT', 0))
tracks_after  = unique_cuts[unique_cuts['POS_ID_CUT'] > my_pos_cut]
winder_start  = tracks_after['WIDTH_ORD_NUM'].sum()
```

**QCS coordinate flip:**  
The winder measures from Drive Side (right), but the QCS scanner measures from Tending Side (left). Positions are flipped:
```python
start_width = deckle_width - winder_end    # QCS Tending Side start
end_width   = deckle_width - winder_start  # QCS Tending Side end
```

---

### 8.7 Time Determination Priority Chain

Accurate GSM calculation requires knowing **when** the jumbo was on the paper machine. Time sources are tried in priority order, most reliable first.

#### End Time Priority Chain

| Priority | Source | Logic |
|---|---|---|
| **0 (Primary)** | `find_end_time_from_remarks()` | Scans `jumbo_ulma_remarks` for operator-entered end time |
| **1** | `ULMA_END` column in `jumbo_ulma_remarks` | Direct end timestamp from ULMA system |
| **1b (sibling fallback)** | Sibling jumbos with same `ULMA_ID` | If this jumbo's `ULMA_END` is NULL, look for another jumbo in the same ULMA group that has it |
| **2** | `ULMATIME + 5 minutes` | ULMA time from `OPTI_DISQUA_11_2_ULMA_TIME`, shifted forward by 5 minutes |
| **3 (last resort)** | `WOUND_TIME` | The winding completion timestamp from `INVENT_DHI` |

#### Start Time Priority Chain

| Priority | Source | Logic |
|---|---|---|
| **0 (Primary)** | Previous ULMA group end | End time of the previous ULMA ID group = start of this jumbo's production window |
| **3 (fallback)** | `ULMATIME − 55 minutes` | Assumes a ~55-minute production window before ULMA timestamp |
| **4 (last resort)** | `WOUND_TIME − 55 minutes` | Same assumption using wound time |

**Sanity check (lines 2312–2324):**  
After both times are determined, a chronological check resets the start time if:
- `start_time >= end_time` (start is after end — clearly wrong)
- `(end_time - start_time).total_seconds() > 90 * 60` (window exceeds 90 minutes — likely wrong pairing)

---

### 8.8 Per-Set Time Narrowing

When a jumbo roll is cut into multiple sets (each set = one pass through the winder), the jumbo's overall time window must be narrowed to only the time window for the specific set being analyzed.

This is done only when `PI_CONNECTED = True` and `set_id` is known.

**Algorithm (lines 2345–2403):**

1. Fetch all sets for the jumbo, sorted by `SET_ID`
2. Calculate cumulative length before each set: `cumulative_before`
3. Calculate the set's fractional position within the jumbo:
   ```python
   start_fraction = cumulative_before / total_jumbo_length
   end_fraction   = (cumulative_before + current_set_length) / total_jumbo_length
   ```
4. Map fractions onto the jumbo's time window:
   ```python
   narrowed_start = start_dt + timedelta(seconds=total_duration * start_fraction)
   narrowed_end   = start_dt + timedelta(seconds=total_duration * end_fraction)
   ```
5. If valid (narrowed_end > narrowed_start), replace the original times and set `SET_TIME_NARROWED = True`

This ensures that for a jumbo with 3 sets, each set gets only the ~⅓ of the PI time range that corresponds to when it was actually being produced.

---

### 8.9 GSM Calculation — PI 600-Point Profile

**Lines 2408–2438**

```python
profile_data, _ = gsm_calculator.fetch_pi_profile_600pt(
    pi_server, start_time_str, end_time_str
)

if profile_data and len(profile_data) >= 10:
    overall_gsm, profiled_gsm, active_zones, _, calc_meta = \
        gsm_calculator.calculate_gsm_from_600pt(
            profile_data, start_width, end_width, deckle_width
        )
```

- Fetches 600 data points from OSIsoft PI Server across the time window
- Requires **at least 10 points** — fewer indicates insufficient PI data coverage
- `profiled_gsm` — GSM averaged only over the reel's width position (start_width → end_width)
- `overall_gsm` — GSM averaged over the entire machine width
- `active_zones` — number of PI sensor zones that fell within the reel's position

If PI is not connected → `STATUS = "PI_DISCONNECTED"`, batch processing is aborted.

---

### 8.10 ULMA Substance GSM

**Lines 2441–2478**  
As a supplementary cross-check, the ULMA scanner's 12-zone substance measurement is also extracted:

```sql
SELECT VAL1, VAL2, ..., VAL12
FROM [PSPDQUALDHI].[dbo].[OPTI_DISQUA_11_2_ULMA_TIME]
WHERE JUMBO_ID = '<parent_id>'
  AND QPARAM = 'SUBSTANCE'
```

**Zone mapping logic:**
- Machine width = 330 mm, divided into 12 equal zones (27.5 mm each)
- Total trim = `machine_width - deckle_width`, split equally on both sides (left trim = `total_trim / 2`)
- A zone overlaps with the reel if: `zone_end_winder > start_width AND zone_start_winder < end_width`
- `ULMA_PROFILED_GSM = mean(overlapping zone values)`

This value is written to the `ULMA_PROFILED_GSM` column in results. It is supplementary — if it fails, the track result is not affected.

---

### 8.11 Result Status Codes

Each processed track gets a `STATUS` field indicating the outcome:

| Status | Meaning |
|---|---|
| `SUCCESS` | GSM calculated successfully — all fields populated |
| `NOT_FOUND` | Track number not found in `INVENT_DHI` |
| `NO_TIME` | Could not determine a valid start/end time from any source |
| `PI_DISCONNECTED` | OSIsoft PI Server is not connected |
| `PI_INSUFFICIENT` | PI returned fewer than 10 data points for the time window |
| `PI_ERROR` | An exception occurred during PI data fetch |
| `ERROR` | Any other unexpected exception during processing |
| `PENDING` | Initial state — track not yet processed |
| `NOT_PROCESSED` | Track was in the fetch list but not processed (e.g. app was stopped) |

---

### 8.12 Progress Display and Summary Metrics

During processing, a live progress bar shows:
```
Processing track 47/183: `456789` (overall 103/183)
```

After completion, five metric cards summarize the results:

```
[ Total Tracks ] [ Used ULMA Time ] [ Used WOUND Time ] [ GSM Calculated ✅ ] [ GSM N/A ❌ ]
```

Below the metrics, a **color-coded visual bar** shows the proportion of time sources used:
- 🟢 Green = tracks where ULMA time was used (most reliable)
- 🟠 Orange = tracks where WOUND_TIME was used (fallback)
- ⚫ Grey = other/unresolved tracks

---

### 8.13 Excel Download

After processing, users can download a two-sheet Excel workbook:

**Sheet 1 — `Batch_GSM_Results`**  
Full results table with all 16 columns per track:
`TRACK_NUM, PARENT_ID, SET_ID, WOUND_TIME, WIDTH_ACT, WGT_SCALED, BSW_ACT, PROFILED_GSM, OVERALL_GSM, ULMA_PROFILED_GSM, END_TIME_SOURCE, START_TIME_SOURCE, PI_POINTS, SET_NARROWED, STATUS, ERROR`

**Sheet 2 — `Summary`**  
A pivot-style key-value summary:

| Metric | Value |
|---|---|
| Date Range Start | `batch_start_dt` |
| Date Range End | `batch_end_dt` |
| Total Tracks | N |
| Used ULMA Time | N |
| Used WOUND Time | N |
| GSM Calculated | N |
| GSM Not Available | N |

**File naming format:**  
`Batch_GSM_YYYYMMDD_HHMM_to_YYYYMMDD_HHMM.xlsx`

---

## Key Database Tables Reference

| Table (full qualified name) | Key Columns | Used For |
|---|---|---|
| `[PSPDQUALDHI].[dbo].[invent_dhi_wound]` | TRACK_NUM, PARENT_ID, SET_ID, CORE_ID, WIDTH_ORD, WIDTH_ACT, LENGTH_LINEAL, WOUND_TIME, WGT_SCALED, BSW_ACT, PROD_MACH_ID, MILL_ID, POS_ID_CUT | Primary winder log — one row per wound reel |
| `[PSPDQUALDHI].[dbo].[INVENT_DHI]` | track_num, parent_id, width_ord, length_lineal, bsw_nominal, prod_mach_id, WOUND_TIME | Reel inventory (Quick Query button target) |
| `[PSPDQUALDHI].[dbo].[OPTI_DISQUA_11_2_ULMA_TIME]` | JUMBO_ID, ULMATIME, MACHINE_ID, QPARAM, VAL1…VAL12 | ULMA timing + 12-zone substance measurements |
| `[PSPDQUALDHI].[dbo].[jumbo_ulma_remarks]` | JUMBO_ID, ULMA_ID, ULMA_END, REMARKS | Operator remarks + ULMA group end times |
| `[PSPDQUALDHI].[dbo].[disprd26_dhi]` | (dispatch records) | Dispatch/delivery tracking |
| `[eCubix_PI_Datalogger].[dbo].[OPTI_Data_Oracle_Log]` | JUMBO_ID, MACHINE_ID, QPARAM, TS_DT | QCS quality parameter logs |
| `sys.databases` | name | SQL Server system table — lists all databases |
| `INFORMATION_SCHEMA.TABLES` | TABLE_SCHEMA, TABLE_NAME, TABLE_TYPE | Schema browsing — list of tables |
| `INFORMATION_SCHEMA.COLUMNS` | COLUMN_NAME, DATA_TYPE, CHARACTER_MAXIMUM_LENGTH, IS_NULLABLE, ORDINAL_POSITION | Schema browsing — table column definitions |

---

## Data Flow Diagram

```
User picks date range
        │
        ▼
SQL: SELECT TRACK_NUM FROM invent_dhi_wound
     WHERE WOUND_TIME BETWEEN start AND end
     AND PROD_MACH_ID = '05' AND MILL_ID = 'BCM'
        │
        ▼
For each TRACK_NUM:
  ┌─────────────────────────────────────────┐
  │                                         │
  │  1. Fetch track from invent_dhi_wound   │
  │     → width, weight, parent_id, set_id  │
  │                                         │
  │  2. Fetch all reels in same set         │
  │     → calculate deckle position         │
  │     → start_width, end_width (QCS view) │
  │                                         │
  │  3. Determine production time window    │
  │     Priority: Remarks → ULMA_END →      │
  │     ULMA+5min → WOUND_TIME              │
  │     Narrow to set fraction if multi-set │
  │                                         │
  │  4. Fetch PI 600-point profile          │
  │     → calculate profiled_gsm for        │
  │        reel's cross-machine slice       │
  │                                         │
  │  5. Fetch ULMA 12-zone SUBSTANCE        │
  │     → calculate ulma_profiled_gsm       │
  │                                         │
  │  6. Save result to checkpoint JSON      │
  └─────────────────────────────────────────┘
        │
        ▼
Display results table + summary metrics
        │
        ▼
Excel download (openpyxl)
```

---

## Helper Functions Reference

These functions are defined **inside** `tab4_sql_explorer()` (nested inner functions):

| Function | Signature | Description |
|---|---|---|
| `_build_checkpoint_path` | `(start_str, end_str) → str` | Builds a filesystem-safe checkpoint file path |
| `_load_checkpoint` | `(path) → dict` | Loads JSON checkpoint, returns `{}` if file not found |
| `_save_checkpoint` | `(path, results_dict) → None` | Writes results dict to JSON checkpoint |
| `_clean_sql_val_batch` | `(val) → str\|None` | Returns `None` for `''`, `'None'`, `'NULL'`, `'null'` strings |
| `_parse_time_safe_batch` | `(val_str) → Timestamp\|None` | Safe datetime parser, handles DD-MM-YYYY and YYYY-MM-DD |
| `_process_single_track_batch` | `(track_num_str) → dict` | Full GSM calculation for one track (Tab 2 logic) |

These external functions are also called from inside the batch processor:

| Function | Module | Description |
|---|---|---|
| `clean_ghost_reels_jumbowide` | (module-level) | Removes duplicate/ghost reel entries from a jumbo's dataset |
| `derive_start_parent_id` | (module-level) | Derives the PARENT_ID of the preceding jumbo for start-time lookup |
| `find_end_time_from_remarks` | (module-level) | Searches `jumbo_ulma_remarks` for Priority 0 end time |
| `find_prev_group_end_time` | (module-level) | Finds end time of previous ULMA group for start time |
| `gsm_calculator.fetch_pi_profile_600pt` | `gsm_calculator` module | Fetches 600-point PI profile for a time range |
| `gsm_calculator.calculate_gsm_from_600pt` | `gsm_calculator` module | Computes overall_gsm and profiled_gsm from profile data |

---

## Error Handling and Edge Cases

| Scenario | Handling |
|---|---|
| Database not connected | `conn is None` → "Run Query" and "Analyze" buttons show `st.error()` |
| PI Server not connected | `PI_CONNECTED = False` → batch aborted with error; individual tracks marked `PI_DISCONNECTED` |
| Write SQL entered by user | `_is_read_only()` blocks it, `st.error("Query blocked")` shown |
| Track not found in DB | `STATUS = "NOT_FOUND"`, track still written to checkpoint and results |
| No time found for track | `STATUS = "NO_TIME"`, included in results with GSM = N/A |
| PI returns < 10 points | `STATUS = "PI_INSUFFICIENT"`, included in results |
| App crash mid-batch | Checkpoint JSON has all completed tracks; re-running resumes automatically |
| Sibling jumbo ULMA lookup | If own jumbo's `ULMA_END` is NULL, queries sibling jumbos with same `ULMA_ID` |
| Chronological mismatch | If `start >= end` or window > 90 min, start is reset and fallback chain continues |
| Single-reel set | Uses `POS_ID_CUT` from full jumbo to reconstruct deckle position |
| ULMA GSM failure | Swallowed silently (`pass`) — batch result is not affected |
| Excel generation error | `st.warning(f"Could not generate Excel: {dl_err}")` — results table still shown |

---

## Usage Guide

### Using the SQL Query Runner

1. Open the dashboard and click the **🗄️ SQL Explorer** tab.
2. To try a quick look at recent data, click one of the three preset buttons (e.g. **Recent Reels (INVENT_DHI)**).
3. To write your own query, type a `SELECT` statement in the text area and click **Run Query**.
4. Results appear as a sortable, scrollable dataframe below the button.
5. To explore the database structure, scroll down to **Available Databases** to see all databases, then use **Browse Tables** and **Browse Columns** to inspect schemas.

### Using the Batch Profiled GSM Analyzer

1. Scroll down within the SQL Explorer tab to the **Batch Profiled GSM Analyzer** section.
2. Set your desired **Start Date**, **Start Time**, **End Date**, and **End Time**.
3. Click **🚀 Analyze All Tracks in Date Range**.
4. A spinner will appear while tracks are fetched, then a progress bar shows per-track processing.
5. When complete, the **Results** section shows a summary (5 metric cards + color bar) and the full results table.
6. Click **📥 Download Results Excel** to save the results locally.

**Resuming after an interruption:**
- If the app is closed or refreshes during processing, the checkpoint file preserves all completed work.
- Simply open the same date range again and click "Analyze" — a message will confirm how many tracks were already done, and processing will resume from where it left off.

**To start fresh:**
- Click **🗑️ Clear Checkpoint** before running to delete the checkpoint file for the current date range.

### Understanding GSM Results

| Column | What to look for |
|---|---|
| `PROFILED_GSM` | The most accurate GSM for this reel's specific cross-machine position |
| `OVERALL_GSM` | Machine-average GSM — useful for comparison against `PROFILED_GSM` |
| `ULMA_PROFILED_GSM` | Independent cross-check from ULMA scanner data |
| `END_TIME_SOURCE` | Quality of the time data: `Remarks` or `ULMA_END` = high confidence; `WOUND_TIME` = fallback only |
| `SET_NARROWED` | `Yes` = time was narrowed to this set's production window for better accuracy |
| `PI_POINTS` | Higher is better — more sensor zones overlapping the reel's position |
| `STATUS` | `SUCCESS` = full data; any other value = partial or failed calculation |
