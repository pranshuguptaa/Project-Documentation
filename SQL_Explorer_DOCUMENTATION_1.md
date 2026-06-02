# SQL Explorer Tab — Technical Documentation

**Project:** `pm5_combined_dashboard.py`  
**Tab Label:** `🗄️ SQL Explorer` (Tab 4)  
**Function:** `tab4_sql_explorer()`  
**Source File:** `pm5_combined_dashboard.py` — Lines 1883–2018  

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture and Dependencies](#architecture-and-dependencies)
3. [Database Connection](#database-connection)
4. [Read-Only Safety Guard](#read-only-safety-guard)
5. [Query Engine — `run_query()`](#query-engine----run_query)
6. [Database and Table Constants](#database-and-table-constants)
7. [UI Layout and Sections](#ui-layout-and-sections)
   - 7.1 [Quick Query Buttons](#71-quick-query-buttons)
   - 7.2 [SQL Query Runner](#72-sql-query-runner)
   - 7.3 [Available Databases Browser](#73-available-databases-browser)
   - 7.4 [Browse Tables](#74-browse-tables)
   - 7.5 [Browse Columns](#75-browse-columns)
8. [Key Database Tables Reference](#key-database-tables-reference)
9. [Error Handling](#error-handling)
10. [Usage Guide](#usage-guide)

---

## Overview

The **SQL Explorer** is Tab 4 of the `pm5_combined_dashboard.py` Streamlit application. It is a self-service SQL environment for operators and engineers to directly query the production SQL Server databases without needing any external tools like SSMS.

It provides:
- **One-click preset queries** for the most common lookups
- **A free-form SQL editor** for ad-hoc queries
- **A built-in schema browser** to explore databases, tables, and columns

All queries are enforced as **read-only** at the code level — no user can accidentally (or intentionally) modify or delete any data.

---

## Architecture and Dependencies

```
pm5_combined_dashboard.py
│
├── Database Layer (pyodbc)
│   ├── init_connection()       — singleton DB connection, @st.cache_resource
│   ├── _is_read_only(query)    — security guard, blocks all DML/DDL
│   └── run_query(query)        — cached query execution, returns (columns, rows)
│
└── Tab 4 UI (tab4_sql_explorer)
    ├── Quick Query Buttons     — 3 one-click preset SELECT queries
    ├── SQL Query Runner        — free-form text area + "Run Query" button
    ├── Available Databases     — queries sys.databases
    ├── Browse Tables           — queries INFORMATION_SCHEMA.TABLES
    └── Browse Columns          — queries INFORMATION_SCHEMA.COLUMNS
```

**Packages used:**

| Package | Purpose |
|---|---|
| `streamlit` | UI framework — widgets, layout, dataframes |
| `pyodbc` | SQL Server ODBC connectivity |
| `pandas` | DataFrame display and column formatting |
| `os` | Reading the connection string file path |

---

## Database Connection

**Function:** `init_connection()` — Lines 60–79

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

- `@st.cache_resource` ensures the connection is created **once per app session** and reused — equivalent to a singleton connection pool.
- The connection string is stored in a plain text file (`connection.txt`) on the server, keeping credentials out of source code.
- **Primary path:** `F:\Env_new\pm5 Run chart\connection.txt` (production Windows server)
- **Fallback path:** `connection.txt` in the same directory as the script
- If the connection fails, `conn` is set to `None` and all sections show a "Database not connected" warning.

---

## Read-Only Safety Guard

**Lines 82–94**

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

**How it works:**

- The query is stripped and uppercased before checking.
- Checks both `startswith` (catches `DELETE FROM ...`) and `" keyword "` (catches inline occurrences like `DROP TABLE` inside a longer string).
- Any blocked query is **rejected before being sent to the database** — it never reaches pyodbc.
- The user sees an `st.error()` message inline where the query was triggered.

This is a defense-in-depth measure. The database user account should also be configured with read-only permissions at the DB level, but this guard provides an additional software-layer barrier.

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
| **Read-only check** | Calls `_is_read_only()` before executing anything |
| **Result caching** | `@st.cache_data(ttl=120)` — identical queries return cached results for **2 minutes**, reducing DB load |
| **Column normalization** | All column names are converted to **UPPERCASE** for consistent downstream access |
| **Return type** | Always returns `(list[str], list[tuple])` — column names and row tuples. Returns `([], [])` on any error |
| **Error display** | Calls `st.error()` in-place so the error appears near the query that triggered it |

---

## Database and Table Constants

**Lines 35–38**

```python
WINDER_LOG_TABLE   = "[PSPDQUALDHI].[dbo].[invent_dhi_wound]"
ULMA_TIME_TABLE    = "[PSPDQUALDHI].[dbo].[OPTI_DISQUA_11_2_ULMA_TIME]"
ULMA_REMARKS_TABLE = "[PSPDQUALDHI].[dbo].[jumbo_ulma_remarks]"
DISPATCH_TABLE     = "[PSPDQUALDHI].[dbo].[disprd26_dhi]"
```

These constants define the main production tables using fully-qualified three-part SQL Server object names (`database.schema.table`). They are used throughout the app when building SQL queries programmatically, ensuring no hardcoded strings are scattered across the code.

---

## UI Layout and Sections

### 7.1 Quick Query Buttons

Three one-click buttons appear at the top of the tab. Each runs a pre-built SELECT query immediately when clicked — no typing required.

---

**Button 1 — Recent Reels (INVENT_DHI)**

```sql
SELECT TOP 10
    track_num, parent_id, width_ord, length_lineal,
    bsw_nominal, prod_mach_id, WOUND_TIME
FROM [PSPDQUALDHI].[dbo].[INVENT_DHI]
ORDER BY WOUND_TIME DESC
```

Shows the 10 most recently wound reels from the production reel inventory. Useful for a quick snapshot of what PM5 has produced most recently.

---

**Button 2 — QCS Data Sample**

```sql
SELECT TOP 10
    JUMBO_ID, MACHINE_ID, QPARAM, TS_DT
FROM [eCubix_PI_Datalogger].[dbo].[OPTI_Data_Oracle_Log]
ORDER BY TS_DT DESC
```

Shows recent records from the QCS (Quality Control System) data logger. `QPARAM` is the quality parameter name (e.g. BASIS_WEIGHT, MOISTURE); `TS_DT` is the timestamp. Useful to check whether QCS data is flowing in real-time.

---

**Button 3 — ULMA Times Sample**

```sql
SELECT TOP 10
    JUMBO_ID, ULMATIME, MACHINE_ID
FROM [PSPDQUALDHI].[dbo].[OPTI_DISQUA_11_2_ULMA_TIME]
ORDER BY ULMATIME DESC
```

Shows the most recent ULMA (unwinding/rewinding machine) operation timestamps per jumbo roll. Useful for verifying that ULMA timing data is up to date.

---

### 7.2 SQL Query Runner

A multi-line text area that accepts any custom SELECT statement. The user types their query and clicks **"Run Query"**.

- Uses `st.text_area()` — supports multi-line, complex queries.
- On button click, passes the text to `run_query()`.
- Any write operation (INSERT, UPDATE, DELETE, DROP, etc.) is silently blocked by the read-only guard — the user sees an error message.
- Results are rendered with `st.dataframe()` — sortable, scrollable, full-width.

**Example query a user might type:**
```sql
SELECT TOP 50
    track_num, parent_id, width_ord, length_lineal, WOUND_TIME
FROM [PSPDQUALDHI].[dbo].[invent_dhi_wound]
WHERE PROD_MACH_ID = '05'
  AND WOUND_TIME >= '2024-07-01'
ORDER BY WOUND_TIME DESC
```

---

### 7.3 Available Databases Browser

Automatically queries the SQL Server instance for all accessible databases:

```sql
SELECT name FROM sys.databases ORDER BY name
```

- Displayed as a list/dataframe so users can see every database the app's DB account has access to.
- The database names shown here are what users can reference in `Browse Tables` (Section 7.4).
- This eliminates the need to ask a DBA "which databases are there?".

---

### 7.4 Browse Tables

The user selects a database from a dropdown and clicks **"Show Tables"**. The app queries:

```sql
SELECT TABLE_SCHEMA, TABLE_NAME, TABLE_TYPE
FROM [<selected_database>].INFORMATION_SCHEMA.TABLES
ORDER BY TABLE_SCHEMA, TABLE_NAME
```

- `TABLE_SCHEMA` — the schema (e.g. `dbo`)
- `TABLE_NAME` — the actual table name
- `TABLE_TYPE` — `BASE TABLE` for real tables, `VIEW` for views

Results are rendered as a dataframe. Users can then pick a table name to inspect further in the column browser.

---

### 7.5 Browse Columns

After a table is selected, clicking **"Show Columns"** queries:

```sql
SELECT
    COLUMN_NAME,
    DATA_TYPE,
    CHARACTER_MAXIMUM_LENGTH,
    IS_NULLABLE
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_NAME = '<selected_table>'
ORDER BY ORDINAL_POSITION
```

- Columns appear in the same order as in the actual table definition (`ORDINAL_POSITION`).
- `DATA_TYPE` shows `varchar`, `int`, `datetime`, etc.
- `CHARACTER_MAXIMUM_LENGTH` shows the max string length for text columns (`NULL` for numeric/datetime).
- `IS_NULLABLE` shows `YES` or `NO`.

This acts as a complete in-app schema viewer — users can understand any table's full structure without needing SSMS, Excel exports, or a DBA.

---

## Key Database Tables Reference

| Table (full qualified name) | Key Columns | Description |
|---|---|---|
| `[PSPDQUALDHI].[dbo].[INVENT_DHI]` | track_num, parent_id, width_ord, length_lineal, bsw_nominal, prod_mach_id, WOUND_TIME | Reel inventory — one row per wound reel |
| `[PSPDQUALDHI].[dbo].[invent_dhi_wound]` | TRACK_NUM, PARENT_ID, SET_ID, CORE_ID, WIDTH_ORD, WIDTH_ACT, LENGTH_LINEAL, WOUND_TIME, WGT_SCALED, BSW_ACT, PROD_MACH_ID, MILL_ID | Primary winder log |
| `[PSPDQUALDHI].[dbo].[OPTI_DISQUA_11_2_ULMA_TIME]` | JUMBO_ID, ULMATIME, MACHINE_ID, QPARAM, VAL1…VAL12 | ULMA operation timestamps |
| `[PSPDQUALDHI].[dbo].[jumbo_ulma_remarks]` | JUMBO_ID, ULMA_ID, ULMA_END, REMARKS | Operator remarks and ULMA group end times |
| `[PSPDQUALDHI].[dbo].[disprd26_dhi]` | (dispatch records) | Dispatch and delivery tracking |
| `[eCubix_PI_Datalogger].[dbo].[OPTI_Data_Oracle_Log]` | JUMBO_ID, MACHINE_ID, QPARAM, TS_DT | QCS quality parameter logs |
| `sys.databases` | name | SQL Server system view — lists all databases |
| `INFORMATION_SCHEMA.TABLES` | TABLE_SCHEMA, TABLE_NAME, TABLE_TYPE | Schema metadata — all tables in a database |
| `INFORMATION_SCHEMA.COLUMNS` | COLUMN_NAME, DATA_TYPE, CHARACTER_MAXIMUM_LENGTH, IS_NULLABLE, ORDINAL_POSITION | Schema metadata — all columns in a table |

---

## Error Handling

| Scenario | Behaviour |
|---|---|
| Database not connected (`conn is None`) | All query buttons and "Run Query" show `st.error("Database not connected")` |
| User types a write statement (INSERT/UPDATE/DELETE/DROP etc.) | `_is_read_only()` blocks it before execution; `st.error("Query blocked")` shown inline |
| SQL syntax error or invalid table name | `run_query()` catches the `pyodbc` exception and displays `st.error(f"Query error: {e}")` |
| Query returns zero rows | `st.dataframe()` renders an empty table — no crash |
| Database with no accessible tables | `Browse Tables` shows an empty dataframe |
| Same query run twice within 2 minutes | `@st.cache_data(ttl=120)` returns the cached result instantly, no DB round-trip |

---

## Usage Guide

### Running a Quick Query

1. Open the dashboard and click the **🗄️ SQL Explorer** tab.
2. Click one of the three preset buttons at the top:
   - **Recent Reels (INVENT_DHI)** — last 10 wound reels
   - **QCS Data Sample** — last 10 QCS quality readings
   - **ULMA Times Sample** — last 10 ULMA operation timestamps
3. The result table appears immediately below the buttons.

### Writing a Custom Query

1. Type your `SELECT` statement in the text area.
2. Click **Run Query**.
3. The result renders as a sortable, scrollable dataframe.
4. Write operations (INSERT, UPDATE, DELETE, DROP, etc.) are blocked automatically.

### Exploring the Database Schema

1. Scroll down to **Available Databases** — this lists every database the app can see.
2. In **Browse Tables**, select a database from the dropdown and click **Show Tables** to see all tables and views in it.
3. In **Browse Columns**, select a table and click **Show Columns** to see every column — its name, data type, max length, and whether it allows nulls.
4. Copy a column name from the results and use it directly in your custom query in the SQL Query Runner above.
