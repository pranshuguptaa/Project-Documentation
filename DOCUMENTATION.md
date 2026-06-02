# Winder Defect Mapping Dashboard — Complete Documentation

> **Real-Time Quality Mapping System for Jumbo Roll Rewinding**
> Tracks defects on a paper machine jumbo roll as it is unwound and cut into finished reels.

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Architecture Overview](#2-architecture-overview)
3. [File & Folder Structure](#3-file--folder-structure)
4. [Installation & Setup](#4-installation--setup)
5. [Configuration Reference (.env)](#5-configuration-reference-env)
6. [Core Concepts](#6-core-concepts)
7. [SQL Database & Remarks Parsing](#7-sql-database--remarks-parsing)
8. [Production Historian (PI Server)](#8-production-historian-pi-server)
9. [State Machine](#9-state-machine)
10. [Polling Loop — The Brain](#10-polling-loop--the-brain)
11. [Defect Trigger & Diameter Capture](#11-defect-trigger--diameter-capture)
12. [REST API Reference](#12-rest-api-reference)
13. [Frontend — Single-Page Dashboard](#13-frontend--single-page-dashboard)
14. [Activity Tracker](#14-activity-tracker)
15. [History File (winder_history.txt)](#15-history-file-winder_historytxt)
16. [Data Export](#16-data-export)
17. [How It All Fits Together — End-to-End Flow](#17-how-it-all-fits-together--end-to-end-flow)
18. [Glossary](#18-glossary)

---

## 1. Project Overview

The **Winder Defect Mapping Dashboard** is a real-time web application that helps quality operators at a paper mill know **exactly which reel a defect landed on** when a jumbo roll is being unwound on the rewinder machine.

### The Problem It Solves

When a jumbo roll of paper (often 10,000–35,000 metres long) is produced on the paper machine, operators record defect positions in a remarks field (e.g., "SPOT @ 15200 MTRS, HOLE @ 22400 MTRS"). When this jumbo is later cut into smaller reels (called *sets*) on the rewinder, the operators need to know:

- Which specific reel (which *set*) contains the defect?
- What was the reel diameter at the exact moment the defect passed through?

Manually calculating this is error-prone. This dashboard automates it by continuously monitoring the rewinder's length counter and automatically mapping each defect to a reel + diameter as it is unwound.

### Key Capabilities

| Capability | Description |
|---|---|
| **Auto-load defects** | Reads defect positions directly from the production SQL database |
| **Real-time mapping** | Detects the exact millisecond a defect is unwound using interpolation |
| **Diameter capture** | Queries the PI historian for reel diameter at the defect crossing moment |
| **Set tracking** | Detects when a set is cut (sawtooth reset) and logs each reel's length |
| **Manual override** | Operators can add or override defects and jumbo length manually |
| **History export** | Downloads a date-filtered Excel report of all mapped defects |

---

## 2. Architecture Overview

```
┌────────────────────────────────────────────────────────────────────┐
│                         Browser (index.html)                       │
│                                                                    │
│  ┌──────────────┐  ┌──────────────────────────────────────────┐   │
│  │  Metrics UI  │  │       Defect Mapping Table               │   │
│  │  Progress Bar│  │  (Pending / Triggering / Mapped / Missed)│   │
│  └──────────────┘  └──────────────────────────────────────────┘   │
│         ▲                        ▲                                 │
│         │   HTTP polling         │                                 │
│         │   GET /api/state       │                                 │
│         │   every 1 second       │   POST /api/load_jumbo         │
│         │                        │   POST /api/add_defect         │
└─────────┼────────────────────────┼───────────────────────────────┘
          │                        │
          ▼                        ▼
┌────────────────────────────────────────────────────────────────────┐
│                          Flask Backend (app.py)                    │
│                                                                    │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │                    State Machine (_machine)                   │ │
│  │  running / jumbo_id / defects / set_log / accumulated_len    │ │
│  └──────────────────────────────────────────────────────────────┘ │
│           ▲                                                        │
│           │ writes                                                 │
│  ┌────────┴────────┐   spawns threads     ┌─────────────────────┐ │
│  │  Polling Loop   │ ───────────────────▶ │  Defect Trigger     │ │
│  │  (daemon thread)│                      │  Set Diam Capture   │ │
│  └────────┬────────┘                      └──────────┬──────────┘ │
│           │ reads length                             │reads diam   │
└───────────┼──────────────────────────────────────────┼────────────┘
            │                                          │
            ▼                                          ▼
  ┌──────────────────┐                    ┌────────────────────────┐
  │  PI Server        │                    │    PI Server           │
  │  ACT_REWIND_      │                    │    ACT_REWIND_DIAMETER │
  │  LENGTH (live)    │                    │    (historical query)  │
  └──────────────────┘                    └────────────────────────┘
            │
            ▼
  ┌──────────────────┐
  │  SQL Server DB   │
  │  jumbo_ulma_     │
  │  remarks table   │
  └──────────────────┘
```

**Summary of data flow:**
1. Operator enters a Jumbo ID in the browser.
2. Flask queries SQL for the jumbo's remarks → parses defect positions and total length.
3. A background daemon thread continuously polls the PI Server for the current rewinder length.
4. When the total unwound length crosses a defect's trigger point, a short-lived thread fires, interpolates the exact timestamp, and queries PI for the reel diameter.
5. The browser polls `/api/state` every second and refreshes the UI.

---

## 3. File & Folder Structure

```
winder_dashboard/
├── app.py                 ← Flask backend — entire application logic
├── templates/
│   └── index.html         ← Single-page frontend (HTML + CSS + JS)
├── tracker/               ← Shared activity tracker library
│   ├── __init__.py
│   ├── core.py
│   ├── flask_tracker.py
│   └── config.py
├── winder_history.txt     ← Flat-file CSV database of mapped defects
├── winder_mock.db         ← SQLite mock database (for development)
├── requirements.txt       ← Python dependencies
├── .env.template          ← Template for environment configuration
└── connection.txt         ← (Optional) SQL Server connection string
```

### Key Files Explained

| File | Purpose |
|---|---|
| `app.py` | The entire Flask backend: config, DB, historian, state machine, polling loop, API routes |
| `templates/index.html` | Self-contained single-page app (CSS + JS embedded in HTML, no build tools) |
| `tracker/` | Shared activity-logging library — logs user sessions to Excel automatically |
| `winder_history.txt` | Append-only CSV log of every successfully mapped defect |
| `winder_mock.db` | SQLite database used in development when there is no SQL Server |
| `.env.template` | Documents every available environment variable with defaults |
| `connection.txt` | Alternative to env var for the SQL Server connection string |

---

## 4. Installation & Setup

### Prerequisites

- Python 3.9+
- Access to OSIsoft PI Server (production) **OR** set `USE_MOCK=true` (development)
- Access to SQL Server `[PSPDQUALDHI]` (production) **OR** use `winder_mock.db`
- `PIconnect` library (only required in production mode)

### Steps

```bash
# 1. Navigate to the project folder
cd winder_dashboard

# 2. Create a virtual environment (recommended)
python -m venv venv
source venv/bin/activate   # Windows: venv\Scripts\activate

# 3. Install dependencies
pip install -r requirements.txt

# 4. Configure environment
cp .env.template .env
# Edit .env to match your environment (see Section 5)

# 5. Run the application
python app.py
# Server starts at http://0.0.0.0:5050
```

### Running in Mock / Development Mode

Set `USE_MOCK=true` in your `.env` file. In this mode:
- The PI historian is simulated — the rewinder length increases at a configurable speed.
- No PI Server connection is required.
- A SQLite file (`winder_mock.db`) is used instead of SQL Server.

---

## 5. Configuration Reference (.env)

All configuration is driven by environment variables (loaded via `python-dotenv` from a `.env` file). **No secrets or IP addresses should be hardcoded.**

| Variable | Default | Description |
|---|---|---|
| `SECRET_KEY` | `winder-dev-key` | Flask session signing key. Change in production. |
| `DB_CONNECTION_STRING` | `sqlite:///winder_mock.db` | pyodbc connection string for SQL Server. In production, set to your MSSQL DSN. |
| `HISTORIAN_TAG_LENGTH` | `Rewinder.ActualLength` | PI tag name for rewinder length (overridden in code for production) |
| `HISTORIAN_TAG_DIAMETER` | `Rewinder.ReelDiameter` | PI tag name for reel diameter (overridden in code for production) |
| `HISTORIAN_API_URL` | `http://localhost:5001/historian` | API URL for historian (not used in current production code — kept for future REST-based historian) |
| `POLL_INTERVAL` | `1.0` | How often (seconds) the polling loop reads the historian |
| `SET_RESET_THRESHOLD` | `500.0` | Minimum drop in metres to count as a sawtooth / set reset |
| `USE_MOCK` | `true` | Set to `false` to use the real PI Server and SQL Server |
| `MOCK_WINDER_SPEED_MPS` | `30.0` | Simulated rewinder speed in metres per second (mock mode only) |
| `MOCK_SET_LENGTH_M` | `3000.0` | Simulated set cut length in metres (mock mode only) |
| `MOCK_JUMBO_INITIAL_DIAMETER_MM` | `1500.0` | Starting reel diameter in mm (mock mode only) |
| `MOCK_JUMBO_CORE_DIAMETER_MM` | `300.0` | Core diameter in mm (mock mode only) |

**Production PI tags (hardcoded in `ProductionHistorian`):**
- Length: `PSPD_BCM_PM4_PLC:Channel1.PM4_Rewinder_PLC.ACT_REWIND_LENGTH_`
- Diameter: `PSPD_BCM_PM4_PLC:Channel1.PM4_Rewinder_PLC.ACT_REWIND_DIAMETER`
- PI Server IP: `10.35.28.240`

---

## 6. Core Concepts

Understanding these four concepts is key to understanding the entire application.

### 6.1 Jumbo Roll & Defect Positions

A **jumbo roll** is the output of the paper machine — a very long roll of paper wound onto a core. Defects are recorded as their distance from the **core (start)** of the roll in metres.

```
Core (0 m) ───────────────────────────── Outer end (30,000 m)
                 ↑ defect at 15,200 m
```

### 6.2 The Rewinder Unwinds from Outside In

The rewinder unwinds the jumbo from the **outer end** toward the core. This is the opposite direction from how defects are recorded.

```
Rewinder unwinds this direction ◄──────────────────

Core (0 m) ───────────────────────────── Outer end (30,000 m)
                 ↑ defect at 15,200 m
```

Therefore, if a jumbo is 30,000 m total and a defect is at 15,200 m:
- The defect will be reached when **30,000 − 15,200 = 14,800 m** has been unwound.
- This is called the **target unwound length** for that defect.

### 6.3 The Sawtooth Pattern (Set Cuts)

The rewinder does not unwind the full jumbo in one go. It cuts the paper into smaller reels called **sets**. After each cut, the rewinder's length counter resets to zero and starts counting the next set.

```
Rewinder length (m):
 3000
  |    /|    /|    /|
  |   / |   / |   / |
  |  /  |  /  |  /  |
  | /   | /   | /   |
  |/    |/    |/    |
  0─────────────────── time
  Set1  Set2  Set3  ...  (sawtooth pattern)
```

The application detects this sawtooth drop (length drops by more than `SET_RESET_THRESHOLD_M`) and accumulates the running total:

```
total_unwound = accumulated_set_length + current_set_length
```

### 6.4 Defect Status Lifecycle

Each defect goes through these states:

```
Pending ──▶ Triggering ──▶ Mapped
                           ↘ Error
Pending ──▶ Missed (if total_unwound already passed the trigger point)
```

| Status | Meaning |
|---|---|
| **Pending** | Defect is loaded; rewinder has not yet reached it |
| **Triggering** | Rewinder just crossed the defect's trigger point; historian query in progress |
| **Mapped** | Diameter successfully captured and recorded |
| **Error** | Historian query failed |
| **Missed** | Defect was added after the rewinder had already passed its trigger point |

---

## 7. SQL Database & Remarks Parsing

### 7.1 Database Table

The application reads from one SQL Server table:

```sql
[PSPDQUALDHI].[dbo].[jumbo_ulma_remarks]
  JUMBO_ID   -- e.g. "PMGS250413001"
  ULMA_ID    -- 8-digit ULMA code e.g. "26041301"
  REMARKS    -- free-text operator notes
```

A second table is queried for reel track information when a set is cut:

```sql
[PSPDQUALDHI].[dbo].[invent_dhi_wound]
  PARENT_ID  -- JUMBO_ID
  SET_ID_NUM -- set number
  UNIT_ID    -- produced reel ID
  TRACK_NUM  -- track position on rewinder
```

### 7.2 Remarks Format

Operators write remarks in free text. The application parses three key pieces of information:

**1. Total Jumbo Length** — three patterns are tried in order:

```
CONSIDER JUMBO LENGTH 30581 MTRS   ← preferred format
/13200M/                            ← older slash format
/32537 MTRS/                        ← older slash format with unit
[first 4-5 digit number] MTR        ← fallback pattern
```

**2. Defect Positions** — split by punctuation delimiters (`.`, `,`, `;`), then match:

```
[Defect Name] @ [position] MTRS    → e.g. "SPOT @ 15200 MTRS"
[Defect Name] --- [position] M     → e.g. "HOLE --- 22400 M"
```

**3. Grade Prefix** — extracted from the first line:
```
A1/26041301/6:23AM/...    → grade prefix = "A1", ULMA ID = "26041301"
B2 B2/26041302/...        → grade prefix = "B2"
```

### 7.3 Sibling Search Fallback

If **both** the total length and defects are missing from a jumbo's own remarks, the application searches for **sibling jumbos** — other jumbos from the same production day:

1. Extract the 6-digit date prefix from the ULMA ID (format `YYMMDD`).
2. Search all jumbos with the same date prefix (and next-day prefix if C shift / night shift).
3. Match siblings by ULMA suffix (last 4 digits) or exact ULMA ID.
4. Use the sibling's parsed defects and length.

> **Design note:** Grade prefix alone (e.g., "B4 == B4") is intentionally **not** used for matching because multiple jumbos per day share the same grade, which would assign the wrong defects.

### 7.4 Remarks Status Values

| Status | Meaning |
|---|---|
| `no_record` | Jumbo ID not found in the database at all |
| `no_remarks` | Row exists but `REMARKS` field is null or blank |
| `clean` | Remarks exist, length parsed, but no defects found (clean run) |
| `has_defects` | Remarks exist with at least one defect parsed |

---

## 8. Production Historian (PI Server)

The `ProductionHistorian` class wraps the **PIconnect** library to read real-time and historical data from an OSIsoft PI Server.

### 8.1 Tags Used

| Tag | Purpose |
|---|---|
| `PSPD_BCM_PM4_PLC:Channel1.PM4_Rewinder_PLC.ACT_REWIND_LENGTH_` | Current rewinder output length (live, sawtooth pattern) |
| `PSPD_BCM_PM4_PLC:Channel1.PM4_Rewinder_PLC.ACT_REWIND_DIAMETER` | Current reel diameter (live and historical) |

### 8.2 Methods

#### `start(jumbo_total_length)`
Connects to PI Server `10.35.28.240`, resolves both tags. Called when a jumbo is loaded.

#### `get_current_length() → (float, datetime)`
Returns the current value of the length tag plus the current UTC timestamp. Called by the polling loop every `POLL_INTERVAL_S` seconds.

#### `get_diameter_at_timestamp(ts: datetime) → float`
Fetches recorded values in a **±2 second window** around `ts` and returns the first valid value in millimetres. Called when a defect is triggered.

#### `interpolate_crossing_time(target_m, prev_total, curr_total, t_prev, t_curr) → datetime` *(static)*
Given two consecutive poll readings that bracket the defect's trigger point, calculates the exact UTC timestamp when the rewinder crossed that point using linear interpolation:

```
frac = (target_m - prev_total) / (curr_total - prev_total)
t_cross = t_prev + frac × (t_curr - t_prev)
```

---

## 9. State Machine

The application maintains a single global dictionary `_machine` that represents the complete current state of the rewinder session. A `threading.Lock` (`_lock`) protects all mutations.

### 9.1 State Dictionary Fields

| Field | Type | Description |
|---|---|---|
| `running` | bool | True while the polling loop is actively monitoring |
| `jumbo_id` | str | Currently loaded jumbo roll ID |
| `jumbo_total_length` | float | Total length of the jumbo in metres |
| `defects` | list[dict] | All defects for this jumbo (pending, mapped, missed) |
| `accumulated_set_length` | float | Sum of all fully wound sets so far (metres) |
| `previous_winder_length` | float | Rewinder length reading from last poll |
| `current_winder_length` | float | Rewinder length reading from this poll |
| `total_jumbo_unwound` | float | `accumulated + current_winder_length` |
| `prev_total_unwound` | float | `total_jumbo_unwound` from last poll (for crossing detection) |
| `set_count` | int | Number of sets cut so far |
| `set_log` | list[dict] | Details of each completed set |
| `t_prev_wall` | datetime | Wall-clock time of the previous poll |
| `remarks_status` | str | `"no_remarks"`, `"clean"`, or `"has_defects"` |
| `error` | str | Last error message (displayed in browser) |
| `is_first_poll` | bool | Skips sawtooth detection on the very first reading |
| `ignore_next_drop` | bool | Skips the first drop if the rewinder was left at high length from a previous jumbo |

### 9.2 Set Log Entry Structure

```json
{
  "set_number": 3,
  "set_length_m": 2984.5,
  "accumulated_total_m": 8943.5,
  "tracks": [
    { "reel_id": "REEL001", "track_num": "1" },
    { "reel_id": "REEL002", "track_num": "2" }
  ],
  "captured_diameter_cm": 87
}
```

### 9.3 Defect Entry Structure

```json
{
  "id": 1,
  "defect_type": "SPOT",
  "position_m": 15200.0,
  "target_unwound_m": 14800.0,
  "captured_diameter_cm": 112,
  "captured_at_ts": "2024-04-13T08:23:14.456789",
  "status": "Mapped"
}
```

---

## 10. Polling Loop — The Brain

The polling loop runs as a **daemon thread** for the lifetime of the application. It executes every `POLL_INTERVAL_S` seconds (default: 1 second).

### Loop Steps

```
┌─────────────────────────────────────────────────────┐
│                  Each iteration:                     │
│                                                      │
│  1. Poll historian → curr_len, t_curr               │
│                                                      │
│  2. First poll? → initialise prev_len, skip          │
│                                                      │
│  3. Sawtooth check:                                  │
│     prev_len > THRESHOLD  AND                        │
│     curr_len < (prev_len - THRESHOLD)                │
│     → accumulated_set_length += prev_len             │
│     → set_count += 1                                 │
│     → query SQL for produced reel track IDs          │
│     → create set_log entry                           │
│                                                      │
│  4. Compute total_unwound = accumulated + curr_len  │
│                                                      │
│  5. Scan defects for crossing:                       │
│     prev_total < target <= curr_total               │
│     → mark as "Triggering"                           │
│     → spawn defect trigger thread                    │
│                                                      │
│  6. Stopping condition:                              │
│     total_unwound >= jumbo_total_length              │
│     → set running = False                            │
└─────────────────────────────────────────────────────┘
```

### Thread Safety

- All state mutations are performed **inside a single `with _lock:` block**.
- Historian I/O (which can take hundreds of milliseconds) happens **before** acquiring the lock.
- Defect trigger threads are spawned **after** releasing the lock to avoid blocking the next poll.

---

## 11. Defect Trigger & Diameter Capture

### 11.1 `_process_defect_trigger()` (per-defect thread)

When the polling loop detects that `total_unwound` has crossed a defect's trigger point, it spawns this function in a new thread with:
- The two bracketing `total_unwound` values (`prev_total`, `curr_total`)
- The two bracketing wall-clock timestamps (`t_prev_wall`, `t_curr_wall`)

**Step 1 — Interpolate crossing timestamp:**
```python
frac = (target_m - prev_total) / (curr_total - prev_total)
t_cross = t_prev_wall + frac × Δt
```

**Step 2 — Query historian for diameter:**
```python
diam_mm = historian.get_diameter_at_timestamp(t_cross)
diam_cm = round(diam_mm / 10.0)   # integer centimetres
```

**Step 3 — Write results:**
- Updates the defect in `_machine["defects"]` with `status="Mapped"`, `captured_diameter_cm`, `captured_at_ts`.
- Appends a row to `winder_history.txt`.

### 11.2 `_capture_set_diameter()` (per-set thread)

When a set cut is detected, this function queries the historian for the reel diameter at the moment the set was cut (using `t_prev_wall` — the last timestamp before the sawtooth drop). The result is written into the set's `set_log` entry as `captured_diameter_cm`.

---

## 12. REST API Reference

The application exposes the following HTTP endpoints:

### `GET /`
Returns the dashboard HTML page (`templates/index.html`).

---

### `POST /api/load_jumbo`
Loads a jumbo roll and initialises the state machine.

**Request body (JSON):**
```json
{
  "jumbo_id": "PMGS250413001",
  "manual_length": "30000"   // optional — overrides length from remarks
}
```

**Response (success):**
```json
{
  "ok": true,
  "jumbo": {
    "jumbo_id": "PMGS250413001",
    "total_length_m": 30581.0,
    "raw_remarks": "B2/26041301/3:03/.../SPOT @ 15200 MTRS...",
    "grade_spec": "B2",
    "remarks_status": "has_defects"
  },
  "defects": [
    {
      "id": 1,
      "defect_type": "SPOT",
      "position_m": 15200.0,
      "target_unwound_m": 15381.0,
      "status": "Pending",
      "captured_diameter_cm": null,
      "captured_at_ts": null
    }
  ]
}
```

**Response (error cases):**
```json
{ "ok": false, "error": "Jumbo 'PMGS...' not found in the database." }
{ "ok": false, "error": "Length could not be found in remarks. Please enter the length manually." }
```

---

### `POST /api/stop`
Pauses the state machine (sets `running = False`). The jumbo data and defect list are preserved.

**Response:** `{ "ok": true }`

---

### `POST /api/add_defect`
Adds a defect manually (not from the database).

**Request body (JSON):**
```json
{
  "defect_type": "HOLE",
  "position_m": "22400"
}
```

**Response:**
```json
{
  "ok": true,
  "already_passed": false,
  "target_unwound_m": 8181.0
}
```

If `already_passed: true`, the defect is immediately marked as `Missed`.

---

### `POST /api/remove_defect`
Removes a defect from the active list.

**Request body (JSON):**
```json
{ "id": "MANUAL-a3b2c1d0" }
```

**Response:** `{ "ok": true }`

---

### `GET /api/state`
Returns the full current machine state as JSON. The browser polls this every 1 second.

**Response:**
```json
{
  "running": true,
  "jumbo_id": "PMGS250413001",
  "jumbo_total_length": 30581.0,
  "raw_remarks": "...",
  "grade_spec": "B2",
  "remarks_status": "has_defects",
  "current_winder_length": 1423.5,
  "accumulated_set_length": 6000.0,
  "total_jumbo_unwound": 7423.5,
  "set_count": 2,
  "set_log": [ ... ],
  "defects": [ ... ],
  "error": null
}
```

---

### `GET /api/export?date=YYYY-MM-DD`
Downloads an Excel file of all mapped defects for the given date.

**Query parameter:** `date` — e.g. `2024-04-13`

**Response:** Excel file download (`Winder_Defects_2024-04-13.xlsx`)

**Columns in the Excel file:**

| Column | Description |
|---|---|
| Jumbo ID | Jumbo roll identifier |
| Defect | Defect type string |
| Position (M) | Defect position from core in metres |
| Target Rewind Length (M) | Length unwound when defect crosses |
| Set Number | Which set the defect fell in |
| Captured Diameter (cm) | Reel diameter at defect crossing moment |
| Timestamp | ISO 8601 UTC timestamp of the crossing |

---

## 13. Frontend — Single-Page Dashboard

The entire UI lives in `templates/index.html` as a self-contained file — HTML, CSS (embedded in `<style>`), and JavaScript (embedded in `<script>`). No build tools or external bundlers are used.

### 13.1 Visual Layout

```
┌──────────────────────────────────────────────────────────────────────┐
│  ● Winder Defect Mapping   Jumbo: PMGS250413001 [B2] · 30581 M       │
│                            [Jumbo ID input] [Length] [Load] [Stop]   │
│                            [Date picker] [Download Excel]            │
├──────────────────────────────────────────────────────────────────────┤
│  ┌──────────────┐ ┌────────────────┐ ┌──────────────┐ ┌──────────┐  │
│  │ Current      │ │ Sets Cut: 2    │ │ Remaining    │ │ Defects  │  │
│  │ Rewinder Len │ │ SET | LEN | Ø  │ │ Jumbo Length │ │ Mapped   │  │
│  │ 1423.5 m     │ │ S2  |2984| 87cm│ │ 23157 m      │ │ 1 of 3   │  │
│  └──────────────┘ └────────────────┘ └──────────────┘ └──────────┘  │
├──────────────────────────────────────────────────────────────────────┤
│  Jumbo Consumption  ████████░░░░░░░░░░░░░░░░░░░ 24.3%               │
│                         ↑ red tick = pending defect                  │
├──────────────────────────────────────────────────────────────────────┤
│  Defect Mapping Table                [Type input] [Pos input] [Add]  │
│  ┌──────────────┬──────┬────────┬──────────┬────────────┬────────┐  │
│  │ Jumbo ID     │ Set  │Defects │ Position │ Diameter   │Actions │  │
│  │ PMGS250413001│ Set 2│SPOT    │15381.0 m │ 112 cm     │[Remove]│  │
│  │ PMGS250413001│ —    │HOLE    │ 8181.0 m │ ——         │[Remove]│  │
│  └──────────────┴──────┴────────┴──────────┴────────────┴────────┘  │
└──────────────────────────────────────────────────────────────────────┘
```

### 13.2 Live Metrics Cards

| Card | Content |
|---|---|
| **Current Rewinder Length** | Live reading from PI (updates every second) |
| **Sets Cut** | Count of sets + a mini table showing each set's length and captured diameter |
| **Remaining Jumbo Length** | `jumbo_total_length − total_jumbo_unwound` |
| **Defects Mapped** | `N of M total defects` |

### 13.3 Progress Bar

A visual bar showing what percentage of the jumbo has been unwound. Each defect is shown as a **tick mark** on the bar:
- **Red tick** = defect is Pending (not yet reached)
- **Green tick** = defect has been Mapped

### 13.4 Defect Mapping Table

The central table shows all defects with:
- Current status badge (colour-coded)
- Set number (resolved from `set_log` using the `getSetForDefect()` function)
- Captured diameter (shown in green when available)
- **Row flashes orange** when the rewinder is within 500 m of a Pending defect's trigger point
- **Remove button** on each row (calls `/api/remove_defect`)

### 13.5 Empty State Messages

When no defects are present, the table shows a contextual message:

| Remarks Status | Icon | Message |
|---|---|---|
| `no_remarks` | 📋 | "No remarks found for this jumbo in the database." |
| `clean` | ✅ | "Clean run — no defects recorded in remarks for this jumbo." |
| Loading | ⏳ | "Waiting for defect data…" |

### 13.6 JavaScript Functions

| Function | Purpose |
|---|---|
| `pollState()` | Fetches `/api/state` and calls `updateState()` |
| `updateState(s)` | Updates all metrics, progress bar, and defect table from state snapshot |
| `syncDefectTable(defects, setLog)` | Incrementally updates the defect table (adds/updates rows without full rebuild) |
| `rebuildTicks(defects, totalLen)` | Rebuilds all tick marks on the progress bar |
| `getSetForDefect(target, setLog)` | Resolves which set a defect falls in (mirrors backend logic exactly) |
| `loadJumbo()` | POSTs to `/api/load_jumbo` |
| `stopJumbo()` | POSTs to `/api/stop` |
| `addDefect()` | POSTs to `/api/add_defect` |
| `removeDefect(id)` | POSTs to `/api/remove_defect` |
| `exportData()` | Navigates to `/api/export?date=...` |
| `applyDefectToRow(row, d)` | Updates diameter cell and row opacity for missed defects |
| `escHtml(str)` | XSS-safe HTML escaping for user-provided strings |

### 13.7 Polling Mechanism

The frontend does **not** use WebSockets. Instead, it polls the REST endpoint every 1 second:

```javascript
setInterval(pollState, 1000);
pollState(); // immediate first call on page load
```

---

## 14. Activity Tracker

The `tracker/` module is a shared library (identical to the one in the blister inspector project) that automatically logs user sessions to an Excel file on the desktop.

### How It Works

```
Browser request
     │
     ▼
Flask before_request hook → records session start (signed cookie for session ID)
     │
     ▼
Flask after_request hook → records page/API access with timestamp
     │
     ▼
EventBuffer (thread-safe in-memory queue)
     │
     ▼ (every 120 seconds or on shutdown)
BatchFlusher daemon thread
     │
     ▼
ExcelWriter (openpyxl)
     │
     ▼
~/Desktop/Tracker/dashboard_activity.xlsx
  └── Sheet: "WinderDashboard"
```

### Initialisation

```python
# In app.py
init_flask_tracker(app, dashboard_name="WinderDashboard")
```

### Shutdown Flush

```python
import atexit
atexit.register(flush_now)  # flush remaining events when app exits
```

---

## 15. History File (winder_history.txt)

This file acts as a persistent, append-only log of every defect that has been successfully mapped. It is a CSV file with a header row written when the file is first created.

### Format

```
Jumbo_ID,Defect_Type,Position_M,Target_Rewind_Length_M,Set_Number,Captured_Diameter_cm,Timestamp
PMGS250413001,SPOT,15200.0,15381.0,5,112,2024-04-13T08:23:14.456789
PMGS250413001,HOLE,22400.0,8181.0,3,87,2024-04-13T07:45:32.123456
```

### Column Descriptions

| Column | Description |
|---|---|
| `Jumbo_ID` | Jumbo roll identifier |
| `Defect_Type` | Operator-entered defect type string |
| `Position_M` | Defect position from the core (metres) |
| `Target_Rewind_Length_M` | Total unwound length at the crossing moment |
| `Set_Number` | Which set the defect fell in (`N/A` if not yet cut) |
| `Captured_Diameter_cm` | Reel diameter at crossing (integer cm) |
| `Timestamp` | ISO 8601 timestamp (local system time) of the crossing |

This file is the data source for the `/api/export` Excel download.

---

## 16. Data Export

Operators can download a date-filtered Excel report:

1. Select a date using the date picker in the header.
2. Click **Download Excel**.
3. The browser calls `GET /api/export?date=YYYY-MM-DD`.
4. The server reads `winder_history.txt`, filters rows by date prefix in the `Timestamp` column, and returns an Excel file.

**Output filename:** `Winder_Defects_YYYY-MM-DD.xlsx`

---

## 17. How It All Fits Together — End-to-End Flow

Here is the complete lifecycle of a single jumbo roll on the dashboard:

```
Step 1: OPERATOR LOADS JUMBO
  ├── Types "PMGS250413001" in the Jumbo ID field and clicks "Load Jumbo"
  ├── Browser POSTs to /api/load_jumbo
  ├── Flask queries SQL: SELECT ULMA_ID, REMARKS WHERE JUMBO_ID = 'PMGS250413001'
  ├── Parses remarks:  total_length=30581 m
  │                    defects: [SPOT@15200m, HOLE@22400m]
  ├── Computes target_unwound_m for each defect:
  │     SPOT: 30581 - 15200 = 15381 m
  │     HOLE: 30581 - 22400 =  8181 m
  ├── Initialises _machine state
  └── Starts historian connection + polling thread

Step 2: REWINDER RUNS (polling loop, every 1 second)
  ├── Poll PI → curr_len=1423.5 m, t_curr=08:15:32
  ├── accumulated=0, total_unwound=1423.5 m
  └── No defect crossing yet

  ... time passes ...

  ├── Poll PI → curr_len=8181.5 m
  ├── total_unwound = 0 + 8181.5 = 8181.5 m
  ├── HOLE trigger: prev_total(8180.2) < 8181.0 <= 8181.5  ✓
  ├── Mark HOLE as "Triggering"
  └── Spawn _process_defect_trigger(HOLE, target=8181.0, ...)

Step 3: DEFECT TRIGGER THREAD (HOLE)
  ├── Interpolate: t_cross = 07:45:32.456 UTC
  ├── Query PI: diameter at 07:45:32 ± 2s → 870 mm
  ├── diam_cm = round(870/10) = 87 cm
  ├── Update _machine["defects"][HOLE] → status=Mapped, diam=87cm
  └── Append to winder_history.txt

Step 4: SET CUT (sawtooth detected)
  ├── Poll PI → curr_len=12.0 m  (was 2984.5 m last poll)
  ├── Drop: 2984.5 > 500 and 12.0 < 2984.5-500  ✓
  ├── accumulated_set_length += 2984.5  (= 2984.5)
  ├── set_count = 1
  ├── Query SQL for produced reel track IDs
  ├── Create set_log entry {set_number:1, set_length_m:2984.5, ...}
  └── Spawn _capture_set_diameter(set=1, t_cut=t_prev_wall)

Step 5: BROWSER REFRESH (every 1 second)
  ├── GET /api/state
  ├── Receives current snapshot
  ├── Updates metrics cards, progress bar, defect table
  └── Shows HOLE as "Mapped" with 87 cm diameter ← operator sees this!

Step 6: JUMBO COMPLETE
  ├── total_unwound >= 30581 m
  └── running = False  (dashboard stops updating)
```

---

## 18. Glossary

| Term | Definition |
|---|---|
| **Jumbo Roll** | The full-length roll of paper produced by the paper machine. Typically 10,000–35,000 m long. |
| **Set** | A shorter finished reel cut from the jumbo by the rewinder. Typically 2,000–6,000 m long. |
| **Rewinder** | The machine that unwinds the jumbo and cuts it into sets. |
| **Sawtooth** | The characteristic pattern of the rewinder's length counter: rises linearly, drops sharply to zero on each set cut. |
| **ULMA ID** | 8-digit production code with format `YYMMDDSS` (year, month, day, sequence). The first 6 digits are the date prefix. |
| **Jumbo ID** | Unique identifier for a jumbo roll in the production database, e.g. `PMGS250413001`. |
| **Remarks** | Free-text field in the SQL database where production operators record defects and jumbo length. |
| **Target Unwound Length** | The total amount of jumbo that must be unwound for a defect to reach the rewinder cut point. = `jumbo_total_length − defect_position_m`. |
| **PI Server** | OSIsoft PI — an industrial time-series historian that records sensor data from production equipment. |
| **PIconnect** | Python library for querying OSIsoft PI Server. |
| **Historian** | Generic term for the PI Server in this codebase. |
| **Crossing** | The moment `total_jumbo_unwound` passes a defect's `target_unwound_m`. |
| **Cross-and-Look-Back** | The technique of using two consecutive poll readings to interpolate the exact timestamp of a crossing. |
| **Diameter** | The outer diameter of the paper reel at the time a defect is wound, captured from the PI tag. |
| **Sibling** | Another jumbo roll produced on the same day (same ULMA date prefix) used as a fallback for remarks data. |
| **Grade Prefix** | Shift + grade code at the start of remarks, e.g. `A1`, `B2`, `C3`. Shift A = 6AM–2PM, B = 2PM–10PM, C = 10PM–6AM. |
| **Clean Run** | A jumbo with a length recorded in remarks but no defects — the paper ran without any noted quality issues. |
| **winder_history.txt** | Append-only CSV log of every successfully mapped defect with timestamp and reel diameter. |
