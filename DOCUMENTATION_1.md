# Blister Inspector — Full Project Documentation

> A computer-vision web application that automatically detects and grades **print blisters** (ridge-like ink defects) on magenta paper strips scanned from a printer.

---

## Table of Contents

1. [What Is This Project?](#1-what-is-this-project)
2. [How It Works — Big Picture](#2-how-it-works--big-picture)
3. [Project Structure](#3-project-structure)
4. [Installation & Running](#4-installation--running)
5. [Core Pipeline (`core/pipeline.py`)](#5-core-pipeline-corepipelinepy)
   - 5.1 [Configuration — `BlisterConfig`](#51-configuration--blisterconfig)
   - 5.2 [Data Classes — `StripResult` & `BlisterResult`](#52-data-classes--stripresult--blisterresult)
   - 5.3 [Detection Pipeline — `BlisterDetector.process()`](#53-detection-pipeline--blisterdetectorprocess)
   - 5.4 [ROI Reprocessing — `BlisterDetector.process_roi()`](#54-roi-reprocessing--blisterdetectorprocess_roi)
6. [Web Server (`app.py`)](#6-web-server-apppy)
   - 6.1 [API Endpoints](#61-api-endpoints)
   - 6.2 [PDF Support](#62-pdf-support)
   - 6.3 [Audit Logging](#63-audit-logging)
7. [Frontend (`templates/` & `static/`)](#7-frontend-templates--static)
   - 7.1 [Upload View](#71-upload-view)
   - 7.2 [Results Dashboard](#72-results-dashboard)
   - 7.3 [Interactive ROI Adjuster (Cropper.js)](#73-interactive-roi-adjuster-cropperjs)
   - 7.4 [Eraser Tool](#74-eraser-tool)
8. [Activity Tracker (`tracker/`)](#8-activity-tracker-tracker)
9. [Batch Processor (`batch_processor.py`)](#9-batch-processor-batch_processorpy)
10. [Visual Debugger (`visual_debugger.py`)](#10-visual-debugger-visual_debuggerpy)
11. [Configuration File (`config.json`)](#11-configuration-file-configjson)
12. [Output Files & Data Storage](#12-output-files--data-storage)
13. [Quality Rating System](#13-quality-rating-system)
14. [Glossary](#14-glossary)

---

## 1. What Is This Project?

Pharmaceutical and industrial printing processes use **magenta paper test strips** to validate print quality. Over time, tiny horizontal ridges called **blisters** appear on the ink surface. These blisters are measured to grade the health of the printer.

This application automates that grading process:

1. A user uploads a scanned image (PNG/JPEG) or PDF of the strip(s).
2. The system automatically finds each strip in the image, straightens it, removes noise, and detects defect ridges.
3. Each strip is assigned a **quality rating**: **1** (best), **3** (medium), or **9** (worst).
4. Results are displayed in an interactive browser-based dashboard. The user can fine-tune the ROI or erase false detections and re-run analysis.
5. The user can submit their own rating alongside the predicted rating for audit purposes.

---

## 2. How It Works — Big Picture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        User uploads image/PDF                       │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                    ┌──────────▼──────────┐
                    │  Flask Web Server   │  app.py
                    │  /upload  /reprocess│
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │  BlisterDetector    │  core/pipeline.py
                    │  .process()         │
                    └──────────┬──────────┘
                               │
         ┌─────────────────────▼─────────────────────┐
         │              Detection Pipeline             │
         │                                             │
         │  1. HSV Masking → find magenta strips       │
         │  2. Deskew (fitLine + warpAffine)           │
         │  3. Content-Aware ROI Crop                  │
         │  4. CIELAB L-channel extraction             │
         │  5. Inpainting (heal pen marks)             │
         │  6. CLAHE contrast enhancement              │
         │  7. Anisotropic Gaussian blur               │
         │  8. Paint line removal                      │
         │  9. Adaptive Threshold → edge map           │
         │ 10. Morphological noise cleanup             │
         │ 11. Connected Component filtering           │
         │ 12. Density / Severity scoring              │
         │ 13. Rating assignment (1 / 3 / 9)           │
         └─────────────────────┬─────────────────────┘
                               │
                    ┌──────────▼──────────┐
                    │   JSON Response     │
                    │  images + metrics   │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │  Browser Dashboard  │  index.html + script.js
                    │  Interactive UI     │
                    └─────────────────────┘
```

---

## 3. Project Structure

```
blister_inspector/
│
├── app.py                  ← Flask web server (entry point)
├── batch_processor.py      ← CLI tool: run pipeline on many images
├── visual_debugger.py      ← CLI tool: show step-by-step pipeline images
├── config.json             ← Tunable algorithm parameters
├── requirements.txt        ← Python dependencies
├── audit_log.csv           ← Accumulated user vs. predicted ratings
├── report.csv              ← Output from batch_processor.py
│
├── core/
│   ├── __init__.py
│   └── pipeline.py         ← Core CV algorithm (BlisterDetector)
│
├── tracker/
│   ├── __init__.py
│   ├── config.py           ← Tracker settings (Excel path, flush interval)
│   ├── core.py             ← Thread-safe event buffer + Excel writer
│   ├── flask_tracker.py    ← Auto-hooks into Flask before/after_request
│   └── streamlit_tracker.py← Streamlit widget wrappers (future use)
│
├── templates/
│   └── index.html          ← Single-page application HTML
│
├── static/
│   ├── style.css           ← All UI styles
│   └── script.js           ← All frontend logic (upload, dashboard, tools)
│
├── mismatches/             ← Auto-saved strip images where user ≠ prediction
├── erasures/               ← Auto-saved before/after images when eraser was used
└── data/                   ← (Not committed) labelled image folders: 1/, 3/, 9/
```

---

## 4. Installation & Running

### Prerequisites
- Python 3.8 or higher
- A terminal / command prompt

### Steps

```bash
# 1. Navigate into the project folder
cd path/to/blister_inspector

# 2. Create a virtual environment
python3 -m venv .venv

# 3. Activate it
source .venv/bin/activate          # Mac/Linux
# .venv\Scripts\activate           # Windows

# 4. Install dependencies
pip install -r requirements.txt
# Key packages installed:
#   opencv-python, numpy, flask, pymupdf (fitz), openpyxl, pandas, matplotlib

# 5. Start the server
python app.py
# Server starts at http://0.0.0.0:9090

# 6. Open in browser
# http://localhost:9090
```

### Alternative Tools

```bash
# Run the visual step-by-step debugger on a single image
python visual_debugger.py path/to/image.png

# Run batch evaluation against labelled data
python batch_processor.py --input data/ --output report.csv
```

---

## 5. Core Pipeline (`core/pipeline.py`)

This is the heart of the project. It contains all the computer vision logic.

---

### 5.1 Configuration — `BlisterConfig`

`BlisterConfig` is a Python `dataclass` holding every tunable parameter. When `config.json` is present on disk, its values override the defaults at startup.

| Parameter | Default | Description |
|---|---|---|
| `roi_lower_h` / `roi_upper_h` | 140 / 175 | HSV hue range for detecting magenta strips |
| `roi_lower_s` / `roi_upper_s` | 40 / 255 | HSV saturation range |
| `roi_lower_v` / `roi_upper_v` | 80 / 255 | HSV value (brightness) range |
| `roi_min_strip_area` | 50,000 px | Minimum contour area to be treated as a strip |
| `roi_morph_kernel` | 15 | Kernel size for morphological closing during strip detection |
| `strict_hsv_lower/upper` | [140,50,80] / [175,255,255] | Tighter HSV range used inside the strip (excludes pen marks) |
| `clahe_clip_limit` | 3.0 | Contrast limiting for CLAHE enhancement |
| `adaptive_block_size` | 51 | Neighbourhood size for adaptive threshold |
| `adaptive_c` | 8 | Constant subtracted in adaptive threshold |
| `min_area_threshold` | 57 px | Minimum blob area — anything smaller is paper grain |
| `density_threshold_low` | 0.5 % | Defect density above this → Rating 3 |
| `density_threshold_high` | 1.40 % | Defect density above this → Rating 9 |
| `severity_threshold_low/high` | 10.0 / 40.0 | Gradient-based severity thresholds (used in `process_roi`) |
| `blister_threshold_low/high` | 24 / 40 | Blister count thresholds (currently unused in scoring) |
| `interactive_roi` | false | Show OpenCV trackbar window to manually adjust crop (desktop only) |

---

### 5.2 Data Classes — `StripResult` & `BlisterResult`

#### `StripResult`
Holds all metrics for **one individual strip** found on the page.

| Field | Type | Description |
|---|---|---|
| `strip_index` | int | Zero-based index of this strip in the image |
| `bounding_box` | tuple(x,y,w,h) | Pixel coordinates of the final analysis ROI |
| `roi_area` | float | Area in pixels of the region of interest |
| `total_blisters` | int | Number of distinct connected blister components |
| `defect_area` | float | Total pixel area occupied by detected defects |
| `defect_density_pct` | float | `defect_area / roi_area × 100` |
| `severity_index` | float | Mean gradient magnitude inside defect regions |
| `quality_rating` | int | 1, 3, or 9 |
| `edges` | ndarray | Binary edge image (white = defect pixels) |
| `img_1` | ndarray | Trimmed colour ROI (for display) |
| `img_4` | ndarray | CLAHE-enhanced greyscale L-channel (for display) |
| `img_8` | ndarray | Annotated final image — grey background, red defects, green circles |
| `strip_bgr` | ndarray | Full rotated strip before ROI crop |
| `trim_box` | tuple | (left, top, right, bottom) trim amounts applied |
| `blister_shapes` | list[dict] | `[{cx, cy, radius}, ...]` — circle coordinates for JS canvas overlay |

#### `BlisterResult`
Aggregated result for the **entire uploaded image**.

| Field | Description |
|---|---|
| `quality_rating` | Rating of the **worst** strip found |
| `defect_density_pct` | Density of the worst strip |
| `total_blisters` | Sum across all strips |
| `total_defect_area` | Sum across all strips |
| `roi_area` | Sum across all strips |
| `strip_results` | List of `StripResult` objects |

---

### 5.3 Detection Pipeline — `BlisterDetector.process()`

Called with a file path, returns a `BlisterResult`. Here is each step explained:

#### Step 1 — Find Magenta Strips (HSV Masking + Morphology)
```
Image → Convert to HSV → Threshold to hue range [140–175] → 
Morphological CLOSE (fill holes) → Find Contours → 
Filter by min area (50,000 px)
```
The HSV range isolates the magenta/pink colour of the test strips. Morphological closing joins broken regions. Only contours above the minimum area threshold are considered actual strips.

#### Step 2 — Deskew Each Strip (Rotation Correction)
```
Fit a line through the strip contour (cv2.fitLine) → 
Calculate deviation from vertical → 
Rotate entire image by that angle (warpAffine, bicubic)
```
This corrects for strips that were placed at a slight angle in the scanner, ensuring perfectly vertical strips for accurate analysis.

#### Step 3 — Content-Aware ROI Crop
```
After straightening: scan each row/column of the strip for 
magenta coverage → find true top, bottom, left, right edges → 
Add safety buffer (3% vertical, 5% horizontal) → 
Enforce minimum crop (12% vertical, 10% horizontal)
```
Instead of blindly cropping fixed percentages, the algorithm measures how "magenta" each row/column is. This ensures pen labels and tape at the ends of the strip are excluded from analysis. A **Precise Magenta Crop** then shaves rows/columns that are less than 30% magenta, followed by an **Aggressive Edge Shave** of 8 pixels on all sides to remove white corner triangles.

#### Step 4 — CIELAB L-Channel Extraction
```
ROI BGR → Convert to CIELAB colour space → Extract L channel (Lightness)
```
CIELAB's L channel is perceptually uniform and separates brightness from colour. Blisters appear as dark horizontal shadows in L.

#### Step 5 — Inpainting (Pen Mark Removal)
```
Strict HSV mask → Invert mask (non-magenta pixels become white) → 
cv2.inpaint (TELEA algorithm, radius=3) → 
Healed L-channel
```
Pen writing, labels, or dirt on the strip creates non-magenta pixels. Inpainting "fills in" these regions using surrounding pixel values, so they don't trigger false detections.

#### Step 6 — CLAHE Enhancement
```
Healed L-channel → CLAHE (clipLimit=3.0, tileGrid=8×8) → 
Enhanced L-channel
```
**Contrast Limited Adaptive Histogram Equalization** locally enhances contrast in 8×8 tile regions, making faint blister shadows clearly visible against the magenta background.

#### Step 7 — Anisotropic Gaussian Blur
```
Enhanced L → GaussianBlur(kernel=(9,1)) → Blurred L
```
A wide horizontal kernel (9 pixels) with only 1 pixel vertically. This blurs away circular paper grain (which is round) but mathematically **preserves** horizontal ridges/blisters (which are elongated horizontally).

#### Step 8 — Paint Line Removal (Pre-Threshold)
```
Invert blurred → Morphological OPEN with wide horizontal kernel (50% strip width) → 
Threshold to binary → Dilate vertically (21px) → 
Inpaint blurred L-channel over paint line mask
```
Printing roller artifacts create continuous horizontal dark bands across the full strip width. This step detects and removes them before thresholding, so they never register as blisters.

#### Step 9 — Adaptive Threshold
```
Blurred L → adaptiveThreshold (GAUSSIAN_C, THRESH_BINARY_INV, 
block=51, C=8) → Edge binary image
```
Adaptive thresholding compares each pixel to its local neighbourhood, making it robust to global lighting variations. `THRESH_BINARY_INV` makes defects (dark regions) white in the output.

#### Step 10 — Morphological Paint Streak Eraser
```
Close horizontal gaps (25×1 kernel) → 
Open with wide kernel (35% strip width × 3px) to extract full-length streaks → 
Dilate vertically (25px) → 
Subtract streak mask from edges (bitwise AND with NOT)
```
A secondary morphological pass to catch any remaining paint streaks that survived the inpainting step.

#### Step 11 — Bridge Broken Blisters
```
Morphological CLOSE (5×5 kernel) on edges
```
Blisters sometimes appear as broken dotted lines. Closing fills small gaps so they are counted as one connected component.

#### Step 12 — Connected Component Filtering
Each blob in the edge image is analysed and **removed** if:
- It touches the top/bottom 15 pixels (tape artifacts)
- Its area < `min_area_threshold` (paper grain)
- It is small (< 35×35 px) and nearly circular (aspect ratio < 2) — pen tap or dirt
- It spans > 40% of strip width AND is thin (height < 100px) — paint streak

#### Step 13 — Scoring
```
defect_area  = total white pixels in final edge image
density_pct  = (defect_area / roi_area) × 100

severity     = mean gradient magnitude (Sobel) at defect pixels
               (measures how deep/sharp the blisters are)

total_blisters = number of connected components

Rating:
  density >= density_threshold_high  → 9  (worst)
  density >= density_threshold_low   → 3  (medium)
  otherwise                          → 1  (best)
```

The page-level rating is the **worst strip rating** found on the page.

---

### 5.4 ROI Reprocessing — `BlisterDetector.process_roi()`

This method is called when the user manually adjusts the crop in the browser. It receives a BGR image (the cropped strip) and an optional **ignore mask** (painted by the Eraser tool).

The pipeline is identical to the strip-level analysis in `process()` — same inpainting, CLAHE, blur, threshold, cleanup, and scoring — but skips the initial strip detection and deskew since the crop is already correct.

The **ignore mask** allows users to paint over false detections in the browser; those pixels are zeroed out in the final edge map before scoring.

---

## 6. Web Server (`app.py`)

Built with **Flask**. Runs on `0.0.0.0:9090`.

```python
detector = BlisterDetector(keep_intermediates=True)  # global singleton
scan_history = []  # last 2 scans for the history panel
```

`keep_intermediates=True` instructs the pipeline to save `img_1`, `img_4`, `img_8` on each `StripResult` for display in the UI.

---

### 6.1 API Endpoints

| Method | Route | Description |
|---|---|---|
| `GET` | `/` | Serve the single-page HTML dashboard |
| `POST` | `/upload` | Accept image/PDF upload, run detection, return JSON |
| `GET` | `/load_demo/<filename>` | Run detection on a pre-loaded demo file |
| `POST` | `/reprocess` | Re-analyse a manually cropped ROI (with optional ignore mask) |
| `GET` | `/history` | Return the last 2 scan results |
| `POST` | `/submit_ratings` | Append user + predicted ratings to `audit_log.csv` |

#### `/upload` — Request
- `multipart/form-data` with field `file`
- Accepts: `.png`, `.jpg`, `.jpeg`, `.bmp`, `.tif`, `.tiff`, `.pdf`
- Max size: 50 MB

#### `/upload` — Response (JSON)
```json
{
  "filename": "scan.pdf",
  "quality_rating": 9,
  "defect_density_pct": 1.85,
  "total_blisters": 42,
  "total_defect_area": 12500.0,
  "roi_area": 675000.0,
  "strips": [
    {
      "strip_index": 0,
      "rating": 9,
      "density": 1.85,
      "severity": 28.4,
      "total_blisters": 42,
      "defect_area": 12500.0,
      "roi_area": 675000.0,
      "img_1": "data:image/png;base64,...",   // colour crop
      "img_4": "data:image/png;base64,...",   // CLAHE greyscale
      "img_8": "data:image/png;base64,...",   // annotated result
      "strip_bgr_full": "data:image/png;base64,...",
      "trim_box": [45, 120, 45, 120],
      "blister_shapes": [{"cx": 100, "cy": 50, "radius": 15}, ...]
    }
  ],
  "history": [...]
}
```

#### `/reprocess` — Request (JSON)
```json
{
  "image_b64": "data:image/png;base64,...",     // cropped strip
  "ignore_mask_b64": "data:image/png;base64,..." // optional eraser mask
}
```

#### `/submit_ratings` — Request (JSON)
```json
{
  "filename": "scan.pdf",
  "strips": [
    {
      "strip_index": 0,
      "user_rating": 3,
      "predicted_rating": 9,
      "density": 1.85,
      "severity": 28.4,
      "total_blisters": 42,
      "used_roi_adjust": true,
      "used_eraser": false,
      "img_1": "data:image/png;base64,...",
      "original_img_8": "...",
      "erased_img_8": "..."
    }
  ]
}
```

---

### 6.2 PDF Support

PDF files are handled using **PyMuPDF** (`fitz`):
1. Open the PDF with `fitz.open()`
2. Render page 0 at **300 DPI** to a pixel map
3. Save as a temporary PNG
4. Pass the PNG path to the detector
5. Clean up the temp files in a `finally` block

---

### 6.3 Audit Logging

Every time the user clicks **"Submit Ratings"**, the following is written to `audit_log.csv`:

| Column | Description |
|---|---|
| Timestamp | `YYYY-MM-DD HH:MM:SS` |
| Filename | Original uploaded filename |
| Strip | Strip index |
| User_Rating | Rating entered by the human operator |
| Predicted_Rating | Rating output by the algorithm |
| Density_% | Defect density percentage |
| Severity | Gradient severity score |
| Total_Blisters | Count of detected blister blobs |
| Used_ROI_Adjust | `Yes` / `No` |
| Used_Eraser | `Yes` / `No` |

Additionally:
- If `user_rating ≠ predicted_rating`, the strip image is saved to `mismatches/` as: `{predicted}_{user}_{filename}_strip{N}.png`
- If the eraser was used, before/after images are saved to `erasures/`.

---

## 7. Frontend (`templates/` & `static/`)

A **single-page application** built with vanilla HTML/CSS/JavaScript. No framework required.

---

### 7.1 Upload View

- **Drag & Drop zone**: accepts PNG, JPEG, PDF
- **Browse Files button**: opens a file picker
- **Demo buttons**: loads pre-configured sample PDFs via `/load_demo/`
- **Recent Scans panel**: shows thumbnails + ratings of the last 2 scans

When a file is selected, `handleFileUpload()` in `script.js` posts the file to `/upload` and shows a spinner overlay while waiting.

---

### 7.2 Results Dashboard

For each strip found, a **strip card** is rendered showing:
- `img_1` — the colour crop of the ROI
- `img_4` — the CLAHE-enhanced greyscale view
- `img_8` — the annotated result (grey background, red defect overlay, green circles)

On top of `img_8`, a JavaScript `<canvas>` draws **green circles** around each detected blister using the `blister_shapes` data from the API response.

The header shows the overall page rating with a colour-coded badge (green = 1, orange = 3, red = 9).

A user rating dropdown is shown for each strip so the operator can record their own assessment.

---

### 7.3 Interactive ROI Adjuster (Cropper.js)

Each strip card has an **"Adjust ROI"** button that opens a modal:

1. The full strip image (`strip_bgr_full`) is loaded into **Cropper.js** (an open-source image cropper library).
2. The initial crop box is pre-positioned using the `trim_box` data.
3. After adjusting, the user clicks **"Apply & Re-analyse"**.
4. The cropped image data URL is sent to `/reprocess`.
5. The strip card is updated in-place with new metrics and images.

The flag `used_roi_adjust` is set to `true` for this strip when ratings are submitted.

---

### 7.4 Eraser Tool

Each strip card also has an **"Erase False Detections"** button:

1. The `img_8` annotated image is loaded onto a canvas.
2. The user paints over false detection circles with a brush.
3. The painted region becomes the **ignore mask** (white pixels = "ignore here").
4. On "Apply Erase", both the cropped strip image and the mask are sent to `/reprocess`.
5. `process_roi()` zeroes out any edge pixels under the mask before scoring.

The original and erased images are tracked and can be saved to `erasures/` when the user submits.

---

## 8. Activity Tracker (`tracker/`)

A lightweight, **framework-agnostic event logging library** that records every user action to an Excel file. It is entirely optional — the CV pipeline does not depend on it.

### Architecture

```
Flask Request
     │
     ▼
flask_tracker.py  (before_request / after_request hooks)
     │ calls
     ▼
core.py → log_event()
     │ pushes to
     ▼
EventBuffer  (thread-safe in-memory list)
     │ drained every N seconds by
     ▼
BatchFlusher  (background daemon thread)
     │ writes to
     ▼
ExcelWriter → dashboard_activity.xlsx
```

### Components

| File | Purpose |
|---|---|
| `config.py` | `EXCEL_PATH` (default: `~/Desktop/Tracker/dashboard_activity.xlsx`) and `BATCH_INTERVAL_SECONDS` (default: 120s). Both overridable via environment variables. |
| `core.py` | `Event` class, `EventBuffer` (mutex-protected list), `ExcelWriter` (openpyxl), `BatchFlusher` (daemon thread). Public API: `log_event()`, `flush_now()`. |
| `flask_tracker.py` | Registers `before_request` (stores start time) and `after_request` (calculates duration, logs action type, path, IP, status code). Session ID is stored in a signed Flask cookie. |
| `streamlit_tracker.py` | Drop-in wrapped versions of Streamlit widgets (`tracked_button`, `tracked_selectbox`, etc.) for future Streamlit dashboards. |

### Excel Output Format

Each row in the Excel sheet represents one HTTP request:

| Timestamp | Session Time | Action Type | Action Detail | Duration (ms) | Extra Data |
|---|---|---|---|---|---|
| 2026-01-15 10:32:05 | 2m 14s | api_call | POST /upload | 1823.4 | `{"ip": "...", "status_code": 200}` |

Each dashboard (app name) gets its own **sheet tab** in the workbook.

### Shutdown Flush

```python
import atexit
atexit.register(flush_now)
```
Ensures any buffered events are written to disk when the server exits.

---

## 9. Batch Processor (`batch_processor.py`)

A command-line tool for evaluating the pipeline's accuracy against a labelled dataset.

### Data Folder Convention

```
data/
├── 1/      ← Images with true rating = 1 (best quality)
├── 1_o/    ← Also rating 1 (alternate folder)
├── 3/      ← True rating = 3
├── 3_o/    
├── 9/      ← True rating = 9 (worst quality)
└── 9_o/    
```

Loose files in the root can also be included if their filename contains `_1`, `_3`, or `_9`.

### Usage

```bash
python batch_processor.py                        # uses data/ folder, outputs report.csv
python batch_processor.py --input /path/to/data  # custom data folder
python batch_processor.py --output results.csv   # custom output file
```

### Output CSV Columns

| Filename | True_Rating | FINAL_RATING | Total_Blisters | Density_Score | Severity_Score |
|---|---|---|---|---|---|

### Terminal Summary

After processing, a summary is printed:

```
=================================================================
  BATCH COMPLETE
=================================================================
  Images processed : 45
  Strips evaluated : 87
  Elapsed time     : 42.31s
  Report saved to  : report.csv

  ┌──────────────────────────────────────────────────┐
  │  PIPELINE ACCURACY: 74/87 = 85.1%               │
  └──────────────────────────────────────────────────┘

  Per-class accuracy:
    Rating 1 (  Best):   92.3%
    Rating 3 (Medium):   78.6%
    Rating 9 ( Worst):   88.9%
=================================================================
```

### PDF Handling
PDFs are automatically converted to PNG at 300 DPI using PyMuPDF, then processed identically to image files. If both `.pdf` and `.png` versions of the same file exist, the `.png` is preferred.

---

## 10. Visual Debugger (`visual_debugger.py`)

A matplotlib-based desktop tool that shows each step of the pipeline as an 8-panel figure. Useful for diagnosing why a particular image is getting an unexpected rating.

### Usage

```bash
python visual_debugger.py path/to/image.png
python visual_debugger.py path/to/image.png --interactive  # OpenCV trackbar ROI adjuster
```

### The 8 Panels

| Panel | Content |
|---|---|
| 1. Trimmed ROI (RGB) | The colour crop sent for analysis |
| 2. CIELAB L-Channel | Greyscale lightness — blisters are dark |
| 3. Strict Masked | L-channel with pen marks inpainted |
| 4. CLAHE Enhanced | Local contrast boosted |
| 5. Gaussian Blur | Anisotropic blur (9×1) |
| 6. Adaptive Threshold | Binary edge detection output |
| 7. Filtered | After noise removal (component filtering) |
| 8. Annotated | Grey strip + red defect pixels + green circles |

The figure title shows the computed **Rating**, **Blister Count**, **Density %**, and **Severity**.

---

## 11. Configuration File (`config.json`)

All algorithm parameters can be changed without touching Python code. Edit `config.json` and restart the server.

```json
{
  "crop_top": 10,
  "crop_bottom": 15,
  "crop_left": 10,
  "crop_right": 28,

  "roi_lower_h": 140,
  "roi_upper_h": 175,
  "roi_lower_s": 40,
  "roi_upper_s": 255,
  "roi_lower_v": 80,
  "roi_upper_v": 255,
  "roi_min_strip_area": 50000,
  "roi_morph_kernel": 5,

  "strict_hsv_lower": [140, 50, 80],
  "strict_hsv_upper": [175, 255, 255],

  "clahe_clip_limit": 3.0,
  "adaptive_block_size": 51,
  "adaptive_c": 7,

  "min_area_threshold": 100,
  "density_threshold_low": 0.289,
  "density_threshold_high": 1.00,

  "texture_min_area": 200,

  "severity_threshold_low": 12.0,
  "severity_threshold_high": 25.0,

  "blister_threshold_low": 24,
  "blister_threshold_high": 40,

  "interactive_roi": false
}
```

**Tuning Tips:**
- If too many false positives (paper grain detected): increase `min_area_threshold`
- If blisters are missed: lower `density_threshold_low` 
- If clean strips are rated 3: raise `density_threshold_low`
- If pen marks are detected: tighten `strict_hsv_lower/upper`
- If strips not found: widen `roi_lower_h` / `roi_upper_h` hue range

---

## 12. Output Files & Data Storage

| File / Folder | Created By | Description |
|---|---|---|
| `audit_log.csv` | `/submit_ratings` API | Accumulated operator ratings vs. predictions |
| `report.csv` | `batch_processor.py` | Batch evaluation results |
| `mismatches/` | `/submit_ratings` | Strip images where prediction ≠ user rating |
| `erasures/` | `/submit_ratings` | Before/after pairs for erased strips |
| `~/Desktop/Tracker/dashboard_activity.xlsx` | `tracker/` | User activity log (page views, API calls) |

---

## 13. Quality Rating System

The project uses a **3-level quality scale** consistent with industry print quality standards:

| Rating | Label | Meaning |
|---|---|---|
| **1** | Best | Clean print. Defect density below threshold. Strip is acceptable. |
| **3** | Medium | Moderate blistering detected. Monitor closely. |
| **9** | Worst | Heavy blistering. Strip is defective. Printer maintenance needed. |

### How the Rating Is Computed

```
defect_density_pct = (total white pixels in final edge image) 
                     ÷ (ROI area in pixels) × 100

if density_pct >= density_threshold_high (default 1.00%):
    rating = 9
elif density_pct >= density_threshold_low (default 0.289%):
    rating = 3
else:
    rating = 1
```

> **Note:** Severity (gradient depth) and blister count thresholds exist in the config and code but are currently only active in the `process_roi()` path. The main `process()` path uses density only.

### Page-Level Aggregation

When multiple strips are found on one page, the page rating equals the **worst strip rating**. This is intentionally conservative — one bad strip fails the page.

---

## 14. Glossary

| Term | Definition |
|---|---|
| **Blister** | A horizontal ridge or ink defect visible on a printed magenta test strip |
| **ROI** (Region of Interest) | The cropped portion of the strip image that is actually analysed |
| **Deskew** | Rotating an image to correct for tilt/angle |
| **CLAHE** | Contrast Limited Adaptive Histogram Equalization — a local contrast enhancement technique |
| **CIELAB** | A colour space where L represents lightness, independent of colour |
| **Inpainting** | Filling in missing/damaged parts of an image using surrounding pixel data |
| **Adaptive Threshold** | A thresholding method that uses local neighbourhood statistics instead of a global cutoff |
| **Morphological Operations** | Image operations based on shape: erosion, dilation, open, close |
| **Connected Component** | A group of connected white pixels treated as one blob/object |
| **Defect Density** | Ratio of defect pixel area to total ROI area, expressed as a percentage |
| **Severity Index** | Mean gradient magnitude at defect locations — measures how sharp/deep the blisters are |
| **Hessian / Frangi Filter** | A ridge-detection filter using second-order derivatives (present in code, currently commented out) |
| **Paint Streak** | A horizontal artifact from the printing roller, spanning the full width of the strip |
| **Ignore Mask** | A user-painted binary mask telling the algorithm to skip specific regions |
| **HSV** | Hue-Saturation-Value colour space — useful for colour-based segmentation |
