# 📊 COPQ Report Dashboard — Plain English Explanation

> **File:** `COPQ_Report_r1_local.pbix` | **Total Pages:** 54 | **Custom Visuals:** 6

---

## 🏭 What Is This Dashboard About?

This is a **Cost of Poor Quality (CoPQ) Report** for a **paper/packaging manufacturing plant** (likely a paper mill with multiple paper machines — PM1A, PM2, PM3, PM4, PM5, PM6, PM7).

**In the simplest terms:**
> Every time the factory produces paper that isn't good enough, or makes more than a customer ordered, or has to redo work — that costs money. This dashboard tracks all those costs, breaks them down by category and machine, compares them against targets, and helps management find where they're losing money due to quality problems.

---

## 🔑 What Does "CoPQ" Mean?

**CoPQ = Cost of Poor Quality** — the money lost because things didn't go right the first time.

Think of it like this: if you baked 10 cakes but 2 burned and 1 was too big, you wasted ingredients + time on those. CoPQ measures that waste — in **Rupees per Tonne (Rs/T)** and in **Tonnes (physical quantity)**.

---

## 📂 The 7 Categories of Quality Loss

![CoPQ Categories breakdown showing 7 types of quality losses](file:///Users/pranshugupta/.gemini/antigravity-cli/brain/d2e94931-1d11-4040-bb4b-f152f2f68498/CoPQ_Categories__600625197822801.png)

| Category | Share of Loss | What It Means (Simply) |
|---|---|---|
| **M/C Stage Repulp** | 47% (biggest!) | Paper made on machines that had to be thrown back & re-processed due to quality issues (GSM/grade changes, machine-specific problems) |
| **Off Grade** | 22% | Paper that didn't meet the required quality spec — blotches, BRI, blister defects |
| **Excess Deliveries** | 11% | Delivering more paper to customers than they ordered (over-production) |
| **Finishing Repulp** | 5% | Paper damaged or rejected at the finishing/converting stage |
| **Transition** | 6% | Losses during changeovers — e.g. switching from Food Grade to Tint |
| **Jumbo Diversion** | 2% | Jumbo rolls diverted due to defects or being out-of-spec |
| **OSS & QSC Quality Repulp** | 7% | Handling defects or sheeting/process defects at quality stages |

> 💡 **Repulp** = the paper got dissolved back into pulp and re-made. It's expensive.
> **Stock lot** = paper that can't be sold as ordered and sits in inventory at a lower value.

---

## 🧮 How Is CoPQ Calculated?

![CoPQ Rs/T Calculation Logic with contribution loss table per machine](file:///Users/pranshugupta/.gemini/antigravity-cli/brain/d2e94931-1d11-4040-bb4b-f152f2f68498/CoPQ_Logic6453622870996614.png)

The dashboard converts physical losses (in Tonnes) into **money (Rs/T)** using:
- **Machine NSR** (Net Sales Realization) — how much revenue each machine generates per tonne
- **Machine-wise Stocklot contribution** — the lower revenue when paper goes to stock-lot
- **Repulp cost ~ Rs. 25,000 per tonne** (fixed assumption)

**Formula (simplified):**  
`CoPQ Rs/T = (Repulp Loss + Stocklot Loss + Excess Delivery Loss + Jumbo Loss) ÷ Total Production Tonnes`

---

## 🗺️ The Approach (Purpose of This Dashboard)

![Project Approach: Identify top contributors, Identify bottlenecks, Visibility via real-time dashboard](file:///Users/pranshugupta/.gemini/antigravity-cli/brain/d2e94931-1d11-4040-bb4b-f152f2f68498/CoPQ_Approach4666503741749368.png)

The dashboard was built with 3 goals:
1. **Identify top 3 contributors** → Repulp, Stock lot, Excess Delivery
2. **Identify the bottlenecks** → Machine-wise analysis + action plans
3. **Visibility** → Real-time CoPQ in Power BI + reports in Opti Vision system

---

## 📑 Page-by-Page Breakdown (54 Pages Total)

### 🏠 Navigation & Context Pages (Pages 1–3, 10–13)

| Page | What It Shows |
|---|---|
| **Home Page** | Landing screen with navigation buttons to 3 sections of the report |
| **CoPQ Categories** | The category breakdown diagram (shown above) |
| **CoPQ Logic** | The calculation methodology & per-machine contribution table |
| **Project Approach** | Why this dashboard was built and the strategy |
| **Excess Del Reasons** | Root cause tree for why excess deliveries happen |
| **Excess Standardised Reports** | Reference images showing how excess is standardized |

---

### 📈 Overall CoPQ Performance Pages (Pages 4–9)

These are the **main KPI pages** — where management sees the big picture.

| Page | What It Shows |
|---|---|
| **CoPQ Baseline** | Overall CoPQ Rs/T vs. the baseline target — are we improving? |
| **CoPQ Trend** | Month-by-month trend charts for each loss category across all machines |
| **CoPQ Rs/T Baseline** | Stacked column chart: each category's Rs/T contribution vs. target; filter by machine |
| **CoPQ Rs/T** | Rs/T breakdown (without compensation), filterable by month & machine |
| **CoPQ Rs/T incl. Compensation** | Same as above but includes customer compensation costs |
| **CoPQ Quantity** | The physical tonnage lost per category per machine (not in money, in Tonnes) |

> 📌 These pages answer: *"How much money are we losing this month vs. last month, and vs. our target?"*

---

### 🏭 Excess Manufacturing Section (Pages 14–18, 30–32, 34)

![Excess Delivery Root Cause Tree — showing manufacturing, logistics, and trim reasons](file:///Users/pranshugupta/.gemini/antigravity-cli/brain/d2e94931-1d11-4040-bb4b-f152f2f68498/CoPQ_Excess_Reasons20505212037007192.png)

Excess Delivery = delivering more paper than a customer ordered. This section deep-dives into why.

| Page | What It Shows |
|---|---|
| **Excess Manufacturing %** | % of orders where we made more than needed, over time |
| **Excess Manufacturing - Categories** | Breakdown by cause: Pattern Edit, Extra Wound, Trim, Weight/Dia variation |
| **Drill Through - Excess Manufacturing** | Order-level detail: each order, how much excess, exact reason |
| **Excess Manufacturing - Analysis** | Pivot tables + pie charts showing grade-wise, machine-wise analysis |
| **Alert - Excess Manufacturing** | Flags orders with excess that need immediate attention |
| **Machine Overview** | How much excess each machine (PM1–PM7) is generating |
| **Month Overview** | Monthly excess manufacturing % trends |
| **Tolerance Comparison** | Comparing planned vs. actual trim tolerances |
| **Excess Comparison** | Year-on-year or period comparison of excess |

![Standardization of Excess Generation — formula: Scaled Qty minus Order Qty = Excess](file:///Users/pranshugupta/.gemini/antigravity-cli/brain/d2e94931-1d11-4040-bb4b-f152f2f68498/CoPQ_Excess_Report15798419294148129.png)

**Key formula:**  
`Excess Delivery = Scaled Qty − (Order Qty + Tolerance)`  
- For **BM** (Basemaker): threshold > 0.35 T
- For **PM** (Paper Machine): threshold > 0.15 T

---

### 📦 Stocklot Section (Pages 19–20)

**Stocklot** = paper that couldn't be delivered to the customer as ordered, so it becomes "stock lot" — sold at a lower price or stored.

| Page | What It Shows |
|---|---|
| **Stocklot** | Total stocklot tonnes over time per machine, % vs. baseline |
| **Stocklot Analysis** | Breakdown by reject reason (REJECTDESC1), grade, and machine |

> 📌 Key metrics: `Hold Qty Tons`, `Stocklot %`, `Stocklot % Baseline`

---

### 🔄 Repulp Section (Pages 21–22, 33)

**Repulp** = paper was dissolved back and re-processed. This is the **biggest cost driver at 47%** of CoPQ.

| Page | What It Shows |
|---|---|
| **Repulp** | Total repulp tonnes over time per machine, % vs. baseline |
| **Repulp Analysis** | Why paper was repulped — reject description, grade, machine |
| **Repulp%** | Pie chart of repulp % by machine |

> 📌 Key metrics: `Slab Qty Tons`, `Repulp %`, `Repulp % Baseline`

---

### 🔩 Machine-Specific Pages (Pages 23–29, 42–44)

One dedicated page per paper machine (PM1A, PM2, PM3, PM4, PM5, PM6, PM7) showing:
- Repulp % trend
- Stocklot % trend
- Excess manufacturing trend
- CoPQ Rs/T contribution from that machine

| Page | What It Shows |
|---|---|
| **PM1A / PM2 / PM3 / PM4 / PM5 / PM6 / PM7 — CoPQ** | Full CoPQ breakdown for each individual machine |
| **Machine Wise Comparison** | All machines side-by-side (repulp, stocklot, excess) |
| **Machine Comparison Rs/T** | Cost in Rs/T compared across all machines |
| **Machine Comparison Tonnage** | Physical loss in tonnes compared across all machines |

---

### 📋 Production & Data Pages (Pages 35–39)

| Page | What It Shows |
|---|---|
| **Production** | Net production per machine per month (the denominator for all % calculations) |
| **Data - CoPQ Rs/T** | Raw data table: Rs/T values per machine per month |
| **Data - CoPQ Qty** | Raw data table: Loss quantities per machine per month |
| **Trim Plan Comparison** | Plan vs. actual trim quantities — was the trim plan followed? |
| **Grade Mix** | How many different grades were produced per machine |

---

### 🎯 Baseline Tracking (Pages 4, 6, 41)

| Page | What It Shows |
|---|---|
| **CoPQ Baseline** | Current CoPQ Rs/T vs. the set baseline target |
| **CoPQ Rs/T Baseline** | Stacked bar showing how each category contributes to the baseline |
| **CoPQ Rs/T Baseline Track - FY 24-25** | Full-year FY2024–25 tracking of whether targets are being met |

---

## 🛠️ Technical Features

| Feature | Detail |
|---|---|
| **Custom visuals used** | Bullet Chart, Candlestick, InfoRiver Charts, Sparklines, Timeline slicer, Tornado Chart, Vertical Bullet Chart |
| **Slicers (filters)** | Month, Machine, Corp/Export flag, Category, Reject Description |
| **Data tables** | Production, Repulp, Stocklot, Receipt_Summary (Excess), TrimSheet, Grade-Mix, CoPQ_cat_Qty |
| **Key calculated measures** | CoPQ Rs/T, Repulp %, Stocklot %, Excess Manufacturing %, Order Completion % |

---

## 🧠 Summary — What This Dashboard Tells You

| Question | Where to Look |
|---|---|
| "How much money are we losing due to quality?" | CoPQ Rs/T Baseline, CoPQ Trend |
| "Which type of loss is biggest?" | CoPQ Categories, CoPQ Quantity |
| "Which machine is the worst performer?" | Machine Wise Comparison, PM-specific pages |
| "Why is paper being sent back (repulped)?" | Repulp Analysis |
| "Why are we delivering more than customers ordered?" | Excess Manufacturing Analysis, Excess Del Reasons |
| "How are we tracking vs. target?" | CoPQ Rs/T Baseline Track FY24-25, Baseline pages |
| "Which orders had problems today?" | Alert - Excess Manufacturing, Drill Through |

---

> [!IMPORTANT]
> **Context:** This is a paper mill's internal quality cost tracking system. The factory has 7 paper machines (PM1A–PM7). Every month, management uses this to find where quality losses are happening, assign responsibility to specific machines, and track whether improvement actions are working.
