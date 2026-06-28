# abhigyansinha_2511978_part4_tableau_dashboard
Tableau Executive Dashboard &amp; Data Storytelling
**Assumptions in Tableau data**
Date fields — one issue. order_date is clean across all 4,200 rows, and delivery_days is perfectly consistent with ship_date − order_date. The issue is with ship_date: 16 orders placed in late December 2025 have a ship_date that falls into January 2026. These are not errors — they are valid multi-day deliveries from year-end orders — but if you ever filter a dashboard by ship_date set to "Year = 2025", those 16 orders will vanish from view. The fix is simple: always filter by order_date, never ship_date.
Geographic fields — two assumptions to document. Rajasthan appears in both the North and West regions, meaning it is split at district level. Aggregating by state without also including region will count Rajasthan twice. More broadly, the dataset has exactly one city per state — city is effectively a label for the state capital, not a true city-level dimension. Do not build city-level analysis expecting more than 20 distinct data points.
Categorical fields — one issue. Five fields (category, sub_category, customer_segment, ship_mode) are fully clean. campaign_channel has 24 nulls. The critical question is whether those nulls mean "direct / untracked" or a data collection failure. Until confirmed, they should be surfaced as "Unknown" in the dashboard, not silently dropped or merged into Organic.
Numerical measures — three things to flag. sales is highly right-skewed (skewness = 2.19) — the mean (₹51,671) is nearly three times the median (₹19,288), pulled up by large Technology orders. Use SUM for totals and MEDIAN for typical-order size. discount is stored as a decimal (0.25 = 25%) and only takes five discrete values — it is not a continuous variable, so building a discount tier dimension will produce cleaner analysis than treating it as a number. customer_rating has 32 nulls — exclude rather than impute; Tableau's AVG function handles this automatically.
Binary and ID fields — one assumption. return_flag is perfectly clean (0/1, no nulls). order_id is fully unique. customer_id has 104 repeat orders from returning customers — meaning COUNTD(customer_id) gives 4,096 unique customers, while COUNT(order_id) gives 4,200 orders. Both are correct, but using the wrong one in the wrong context will silently undercount or overcount. Always be explicit about which you intend.
**Calculated Fields**
**Dashboard:** Executive Sales Dashboard  
**Data source:** `dashboard_sales_data.xlsx`  
**Total calculated fields:** 8  
**Purpose:** Extend the raw dataset with business metrics that cannot be answered by a single column alone

---

## How to create a calculated field in Tableau

Before reading the field definitions below, here is how to add any of these fields in Tableau Desktop:

1. In the **Data** pane (left sidebar), right-click on any blank space
2. Select **Create Calculated Field…**
3. Type the field name in the top box (exactly as shown below)
4. Paste the formula into the editor
5. Check that the status bar at the bottom says **The calculation is valid**
6. Click **OK**

The new field appears in the Data pane under **Measures** (for numbers) or **Dimensions** (for strings). Drag it onto any shelf — Rows, Columns, Colour, Label, or Filters — like any other field.

---

## Calculated Field 1 — Profit Margin (%)

### Formula

```
SUM([profit]) / SUM([sales]) * 100
```

### What it calculates

Profit margin measures what percentage of every rupee of revenue the business keeps as profit after costs. It is calculated at the aggregate level — using `SUM()` rather than row-level division — because dividing individual rows and then averaging them would give a distorted result (a ₹100 order and a ₹1,00,000 order would be weighted equally).

### Why this formula, not `[profit] / [sales]`

Without `SUM()`, Tableau divides profit by sales for each individual row and then aggregates. This produces an average of margins rather than the true aggregate margin. For example:

| Order | Sales | Profit | Row-level margin |
|---|---|---|---|
| A | ₹100 | ₹10 | 10% |
| B | ₹1,000 | ₹200 | 20% |
| Average of row margins | | | 15% |
| **Correct aggregate margin** | **₹1,100** | **₹210** | **19.1%** |

The `SUM()` version gives 19.1% — the number that matches what an accountant would calculate.

### Results in this dataset

| View | Profit Margin |
|---|---|
| Overall | 15.3% |
| Technology | 18.2% |
| Furniture | 6.9% |
| Office Supplies | 14.9% |
| No discount orders | 19.0% |
| 35% discount orders | −4.4% (loss-making) |

### Business meaning

A margin of 15.3% means the business retains ₹15.30 for every ₹100 of revenue after covering the cost of goods. When this number falls below 10%, an order or category is contributing very little to profitability. When it goes negative (as with heavily discounted orders at 35% off), the business is actively losing money on each sale.

### How to use in the dashboard

Drag `Profit Margin (%)` to **Colour** on a regional or category bar chart. Set the colour to a diverging palette (red → white → green), centred at 0. Stores or categories shown in red are loss-making; green indicates healthy margins.

---

## Calculated Field 2 — Cost

### Formula

```
SUM([sales]) - SUM([profit])
```

### What it calculates

Cost represents the total expenditure incurred to generate the reported sales — including the cost of goods sold, shipping, and any other variable costs absorbed before the profit line. The dataset does not store cost directly; it is derived by subtracting profit from sales.

### Assumption

This formula assumes that `sales − profit = cost` holds at the order level in this dataset. The audit confirmed that no order has a cost below zero (i.e. profit never exceeds sales), so this derivation is valid across all 4,200 records.

### Results in this dataset

| Metric | Value |
|---|---|
| Total cost | ₹18.37 Cr |
| Total sales | ₹21.70 Cr |
| Total profit | ₹3.33 Cr |
| Cost as % of sales | 84.7% |
| Minimum cost (single order) | ₹69.05 |
| Maximum cost (single order) | ₹3,65,719 |

### Business meaning

Understanding cost alongside sales reveals where the business is spending most to generate revenue. High cost-to-sales ratios indicate either low-margin products or high-discount orders. Categories with a cost ratio above 90% (cost/sales > 0.9) should be reviewed for pricing or supplier renegotiation.

### How to use in the dashboard

Use `Cost` alongside `Sales` and `Profit` in a stacked bar chart to visualise the cost structure by category or region. A tooltip showing all three values (sales, cost, profit) on a single chart gives leadership the full picture without needing multiple charts.

---

## Calculated Field 3 — Average Order Value (AOV)

### Formula

```
SUM([sales]) / COUNTD([order_id])
```

### What it calculates

Average Order Value (AOV) measures the typical revenue per transaction. It is calculated by dividing total sales by the number of distinct orders — `COUNTD([order_id])` rather than `COUNT([order_id])` — to guard against any future duplicate records.

### Why not `AVG([sales])`

`AVG([sales])` computes the average at the row level and is equivalent to `SUM([sales]) / COUNT([sales])`. In most cases this gives the same answer. However, `SUM([sales]) / COUNTD([order_id])` is more robust because:

- It explicitly counts unique orders, making it immune to duplicate rows
- It aggregates correctly when Tableau applies level-of-detail context (e.g. when the view groups by region, the denominator automatically becomes orders within that region)

### Results in this dataset

| Segment / Channel | AOV |
|---|---|
| Overall | ₹51,671 |
| Referral channel | ₹54,111 (highest by channel) |
| Social channel | ₹53,261 |
| Paid channel | ₹47,245 (lowest by channel) |
| Consumer segment | ₹53,053 |
| Corporate segment | ₹50,487 |
| Home Office segment | ₹51,522 |

### Business meaning

A higher AOV means customers are spending more per transaction. Channels with a high AOV (Referral, Social) are attracting higher-value customers even if their order volume is lower. Improving AOV through cross-selling, bundling, or minimum-order thresholds can grow revenue without requiring more customers.

### How to use in the dashboard

Show AOV on a horizontal bar chart broken down by `campaign_channel`, sorted descending. This immediately surfaces which channels bring in the highest-value orders — a more useful metric than total revenue for evaluating marketing channel quality.

---

## Calculated Field 4 — Return Rate (%)

### Formula

```
SUM([return_flag]) / COUNTD([order_id]) * 100
```

### What it calculates

Return Rate measures the percentage of orders that were returned. `return_flag` is a binary field (1 = returned, 0 = not returned), so `SUM([return_flag])` counts the total number of returns. Dividing by the number of distinct orders gives the return rate as a percentage.

### Why this formula works

Because `return_flag` is always 0 or 1, `SUM([return_flag])` is equivalent to counting all rows where `return_flag = 1`. This is more efficient than writing a `COUNTIF`-style formula and produces the same result.

An alternative formula that is equally valid:
```
COUNTD(IF [return_flag] = 1 THEN [order_id] END) / COUNTD([order_id]) * 100
```

This version explicitly counts only order IDs where a return occurred, which is marginally more readable but computationally identical.

### Results in this dataset

| Breakdown | Return Rate |
|---|---|
| Overall | 4.55% |
| Furniture | 7.67% — highest |
| Office Supplies | 3.65% |
| Technology | 3.03% — lowest |
| East region | 4.91% |
| West region | 4.34% — lowest |

### Business meaning

A 4.55% return rate means roughly 1 in 22 orders is returned. Furniture's rate of 7.67% — more than double Technology's 3.03% — signals a category-level quality, fit, or expectation problem. Returns are expensive: each return costs the business the revenue and typically a portion of the shipping and restocking cost. Tracking return rate by category, region, and campaign channel helps identify where intervention would have the highest impact.

### How to use in the dashboard

Add `Return Rate (%)` as a secondary metric alongside `Profit Margin (%)` on the regional performance chart. Use a dual-axis view or a reference line to show the estate average (4.55%). Regions or categories above the reference line warrant investigation.

---

## Calculated Field 5 — Shipping Delay Bucket

### Formula

```
IF [delivery_days] = 0 THEN "Same Day (0 days)"
ELSEIF [delivery_days] <= 2 THEN "Fast (1–2 days)"
ELSEIF [delivery_days] <= 5 THEN "Standard (3–5 days)"
ELSEIF [delivery_days] <= 7 THEN "Slow (6–7 days)"
ELSE "Very Slow (8+ days)"
END
```

### What it calculates

Shipping Delay Bucket converts the continuous `delivery_days` integer into a meaningful categorical label. Instead of showing a number (e.g. 4 days), it groups delivery performance into five tiers that are immediately interpretable by a non-technical audience.

### Why bucket rather than use delivery_days directly

`delivery_days` ranges from 0 to 9 — ten possible values. On a chart, ten separate bars or colour bands are hard to read. Five named buckets reduce cognitive load and map more naturally to how logistics teams think about performance (same day, fast, standard, slow, very slow).

### Bucket design rationale

| Bucket | Range | Rationale |
|---|---|---|
| Same Day | 0 days | Matches `ship_mode = Same Day` exactly — 317 orders |
| Fast | 1–2 days | Matches `ship_mode = First Class` typical range (mean 1.8 days) |
| Standard | 3–5 days | Covers the majority of orders (2,865 of 4,200) — the typical experience |
| Slow | 6–7 days | Flagged for review — above the Standard Class mean of 4.7 days |
| Very Slow | 8–9 days | Only 45 orders — potential operational failures or remote delivery locations |

### Results in this dataset

| Bucket | Orders | % of total |
|---|---|---|
| Same Day (0 days) | 317 | 7.5% |
| Fast (1–2 days) | 886 | 21.1% |
| Standard (3–5 days) | 2,331 | 55.5% |
| Slow (6–7 days) | 621 | 14.8% |
| Very Slow (8+ days) | 45 | 1.1% |

### Business meaning

> **One-line summary:** An interactive Tableau dashboard answering eight business questions about retail sales performance across 4,200 orders, 20 states, 4 regions, and 3 product categories — built on two years of transactional data (Jan 2024 – Dec 2025).

---

## Table of Contents

1. [Business Problem Summary](#1-business-problem-summary)
2. [Dataset Description](#2-dataset-description)
3. [Tableau Workbook Description](#3-tableau-workbook-description)
4. [Calculated Fields Created](#4-calculated-fields-created)
5. [Dashboard Components](#5-dashboard-components)
6. [Filters and Interactions Used](#6-filters-and-interactions-used)
7. [Key Business Insights](#7-key-business-insights)
8. [Dashboard Story Summary](#8-dashboard-story-summary)
9. [Assumptions and Limitations](#9-assumptions-and-limitations)

---

## 1. Business Problem Summary

A national retail business operates across four geographic regions and four store formats, selling products across three categories: Furniture, Office Supplies, and Technology. Leadership had total sales and profit figures but lacked a structured view of **what was driving performance variation** across regions, product lines, customer segments, and shipping modes.

The core business questions this dashboard was designed to answer:

| # | Business question | Chart type used |
|---|---|---|
| 1 | How are sales changing over time? | Line chart |
| 2 | Which region performs best? | Horizontal bar chart |
| 3 | Which categories drive profit or loss? | Horizontal bar chart (sorted by profit) |
| 4 | How does discount relate to profit? | Scatter plot |
| 5 | Which segment has higher return risk? | Bar chart with reference line |
| 6 | Which shipping mode causes delays? | Stacked bar chart |
| 7 | What is the overall business health? | KPI summary cards |
| 8 | How do filters change the view? | Interactive filter interaction panel |

The intended audience is executive leadership and regional directors who need actionable insights without statistical jargon.

---

## 2. Dataset Description

**File:** `data/dashboard_sales_data.xlsx`
**Period:** January 2024 – December 2025
**Records:** 4,200 orders (320 rows in the regression dataset; full 4,200 here)
**Sheets:** 2 — `dashboard_sales_data` (data) and `data_dictionary` (field reference)

### Field inventory

| Field | Type | Range / Values | Role in dashboard |
|---|---|---|---|
| `order_id` | String | DB-2024-500001 to DB-2025-504200 | Unique order identifier (COUNTD for order counts) |
| `order_date` | Date | 1 Jan 2024 – 31 Dec 2025 | X-axis for trend chart; Year and Month filter |
| `ship_date` | Date | 3 Jan 2024 – 6 Jan 2026 | Used to derive `delivery_days` |
| `customer_id` | String | C-NNNNN (4,096 unique) | Repeat vs new customer analysis |
| `customer_segment` | Categorical | Consumer, Corporate, Home Office | Filter; return risk bar chart X-axis |
| `region` | Categorical | East, North, South, West | Filter; regional bar chart |
| `state` | Categorical | 20 Indian states | State-level bar chart; geographic drill-down |
| `city` | Categorical | 20 cities (one per state) | Context label only |
| `category` | Categorical | Furniture, Office Supplies, Technology | Filter; colour encoding in sub-category chart |
| `sub_category` | Categorical | 13 values | Y-axis of sub-category profitability chart |
| `product_name` | String | High cardinality | Detail view only — not used as dimension |
| `ship_mode` | Categorical | First Class, Same Day, Second Class, Standard Class | Filter; shipping delay stacked bar X-axis |
| `sales` | Float | ₹72.91 – ₹4,27,418 | Primary measure; right-skewed (skew = 2.19) |
| `quantity` | Integer | 1 – 10 | Volume measure |
| `discount` | Float | 0.00 – 0.35 (8 discrete tiers) | Scatter plot X-axis |
| `profit` | Float | −₹17,758 – ₹1,33,628 | Profitability measure; 288 negative values |
| `return_flag` | Binary | 0 or 1 | Return rate calculation |
| `delivery_days` | Integer | 0 – 9 | Shipping delay bucket basis |
| `customer_rating` | Float | 2.1 – 5.0 (32 missing) | Not used in dashboard (r = −0.03 with sales) |
| `campaign_channel` | Categorical | Email, Organic, Paid, Referral, Social (24 missing) | Filter; channel analysis |

### Data quality notes

| Issue | Detail | Resolution |
|---|---|---|
| `ship_date` spills into 2026 | 16 orders placed Dec 2025 shipped in Jan 2026 | Always filter by `order_date`, not `ship_date` |
| `customer_rating` has 32 nulls | 0.8% missing | Excluded from dashboard — near-zero sales correlation |
| `campaign_channel` has 24 nulls | 0.6% missing | Treated as "Unknown" in channel analysis |
| Rajasthan in two regions | Appears in both North and West | Always use region + state together; never state alone |
| `discount` is a decimal (0.25 = 25%) | Not a percentage | Multiply by 100 for display; use Discount Tier field |

---

## 3. Tableau Workbook Description

**File:** `tableau/executive_dashboard.twbx`
**Type:** Tableau Packaged Workbook (contains both workbook XML and embedded data)
**Tableau version:** Desktop 2024.x or later

### Workbook structure

| Sheet name | Chart type | Primary purpose |
|---|---|---|
| Sales Trend View | Line chart | Monthly revenue trajectory over 24 months |
| Regional Performance | Horizontal bar | Sales by region with margin tooltip |
| State Performance | Horizontal bar | Top 10 states by sales (drills down from region) |
| Category Profitability | Horizontal bar | Sub-category profit sorted descending |
| Segment Return Risk | Bar + reference line | Return rate % by customer segment |
| Shipping Delay Profile | Stacked bar | Order distribution across delay buckets by ship mode |
| Discount vs Profit | Scatter plot | Relationship between discount tier and profit margin |
| KPI Summary | Text / BAN tiles | Total sales, profit, margin, orders, return rate |
| Executive Dashboard | Dashboard canvas | Assembly of all sheets with filters |

### How to open

1. Download `executive_dashboard.twbx` from the `tableau/` folder
2. Double-click to open in Tableau Desktop (free trial at tableau.com if needed)
3. All data is embedded — no external connection required
4. All calculated fields are pre-built and ready to use

---

## 4. Calculated Fields Created

Eight calculated fields were created to support the dashboard analysis. All are built directly in Tableau using the fields available in `dashboard_sales_data.xlsx`.

### Field 1 — Profit Margin (%)

```
SUM([profit]) / SUM([sales]) * 100
```

**Purpose:** Measures the percentage of revenue retained as profit after costs.
Uses `SUM()` at the aggregate level — dividing row-by-row and averaging would weight a ₹100 order equally to a ₹1,00,000 order, producing an incorrect blended average.
**Result:** Overall 15.3%; Technology 18.2%; Furniture 6.9%.

---

### Field 2 — Cost

```
SUM([sales]) - SUM([profit])
```

**Purpose:** Derives total cost of goods since the dataset does not store cost directly.
Validated: no row has profit exceeding sales, so cost is always positive.
**Result:** Total cost = ₹18.37 Cr (84.7% of sales).

---

### Field 3 — Average Order Value (AOV)

```
SUM([sales]) / COUNTD([order_id])
```

**Purpose:** Revenue per transaction. Uses `COUNTD` (count distinct) rather than `COUNT` to guard against any duplicate records, and correctly adjusts the denominator when filters are applied.
**Result:** Overall ₹51,671; Referral channel ₹54,111 (highest).

---

### Field 4 — Return Rate (%)

```
SUM([return_flag]) / COUNTD([order_id]) * 100
```

**Purpose:** Percentage of orders returned. `SUM([return_flag])` works because the field is binary (0 or 1), making the sum equivalent to a count of returns.
**Result:** Overall 4.55%; Furniture 7.67% (highest); Technology 3.03% (lowest).

---

### Field 5 — Shipping Delay Bucket

```
IF [delivery_days] = 0 THEN "Same day (0 days)"
ELSEIF [delivery_days] <= 2 THEN "Fast (1–2 days)"
ELSEIF [delivery_days] <= 5 THEN "Standard (3–5 days)"
ELSEIF [delivery_days] <= 7 THEN "Slow (6–7 days)"
ELSE "Very slow (8+ days)"
END
```

**Purpose:** Converts the continuous `delivery_days` integer into five named performance tiers for the stacked bar chart. Boundaries were designed to align with each ship mode's typical delivery profile.
**Sort order in Tableau:** Set manually (right-click → Sort → Manual) to Same day → Fast → Standard → Slow → Very slow. Alphabetical default would reverse the traffic-light logic.

---

### Field 6 — Discount Tier

```
IF [discount] = 0 THEN "No discount (0%)"
ELSEIF [discount] <= 0.10 THEN "Low (1–10%)"
ELSEIF [discount] <= 0.20 THEN "Medium (11–20%)"
ELSEIF [discount] <= 0.30 THEN "High (21–30%)"
ELSE "Very high (31–35%)"
END
```

**Purpose:** Converts the raw decimal discount into readable business tiers. The `discount` field has only 8 discrete values (0, 0.05, 0.10, 0.15, 0.20, 0.25, 0.30, 0.35) — treating it as continuous is misleading.
**Result:** Very high tier (31–35%) averages −4.4% margin — loss-making.

---

### Field 7 — Net Revenue After Discount

```
[sales] * (1 - [discount])
```

**Purpose:** Confirms what-if analysis. Note that `sales` in this dataset already reflects post-discount revenue (the amount the customer paid), so this field is used for scenario modelling rather than as the primary revenue measure.

---

### Field 8 — Customer Type (New vs Repeat)

```
IF { FIXED [customer_id] : COUNTD([order_id]) } > 1
THEN "Repeat customer"
ELSE "New customer"
END
```

**Purpose:** Classifies each order as belonging to a new or returning customer using a Level of Detail (LOD) expression. The `FIXED` keyword ensures the count of orders per customer is computed across the entire dataset, not within the current view's filter context.
**Result:** 104 of 4,096 unique customers placed more than one order (2.5% repeat rate).

---

## 5. Dashboard Components

### KPI cards (summary tiles)

Five KPI cards sit at the top of the dashboard and update with every filter change:

| KPI | Formula | Full-dataset value | Colour signal |
|---|---|---|---|
| Total sales | `SUM([sales])` | ₹21.70 Cr | Neutral blue |
| Total profit | `SUM([profit])` | ₹3.33 Cr | Green if margin ≥ 15.3%, red if below |
| Profit margin | `Profit Margin (%)` | 15.3% | Green ≥ 15.3%, red below |
| Total orders | `COUNTD([order_id])` | 4,200 | Neutral |
| Return rate | `Return Rate (%)` | 4.55% | Green ≤ 4.55%, amber 4.55–6%, red above 6% |

### Charts

| Chart | Sheet | Fields used | Why this type |
|---|---|---|---|
| Monthly sales trend | Sales Trend View | `order_date` (Month), `SUM(sales)` | Continuous time series — line shows direction |
| Regional sales | Regional Performance | `region` (sorted), `SUM(sales)` | Named categories with text labels — horizontal bar |
| State top 10 | State Performance | `state`, `region`, `SUM(sales)` | Drill-down from region; same rationale |
| Sub-category profit | Category Profitability | `sub_category`, `SUM(profit)`, `category` (colour) | 13 items sorted by profit — bar not pie |
| Return risk | Segment Return Risk | `customer_segment`, `Return Rate (%)`, reference line at 4.55% | 3 groups on one % measure + diagnostic line |
| Shipping delays | Shipping Delay Profile | `ship_mode`, `COUNTD(order_id)`, `Shipping Delay Bucket` (stack colour) | Volume + composition simultaneously |
| Discount scatter | Discount vs Profit | `discount` (X), `profit margin %` (Y), `category` (colour) | Relationship between two continuous measures |

---

## 6. Filters and Interactions Used

### Active filters on the dashboard

Six filters appear on the dashboard and are applied to all worksheets simultaneously via **Apply to Worksheets → All Using This Data Source**:

| Filter | Field | Control type | What it changes |
|---|---|---|---|
| Year | `order_date` (Year) | Single-value dropdown | Trend chart collapses to 12 months; all KPIs scale |
| Region | `region` | Multiple values list | State chart shows only states in selected region |
| Category | `category` | Button (3 options) | Sub-category chart filters to selected parent |
| Customer segment | `customer_segment` | Checkboxes | All charts filter to segment orders |
| Ship mode | `ship_mode` | Multiple values list | Shipping delay chart highlights selected mode |
| Campaign channel | `campaign_channel` | Dropdown | All charts filter to channel orders |

### Filter interactions

**Action filter — click a region bar to filter state chart:**
In Tableau Desktop: Dashboard → Actions → Add Action → Filter.
Source sheet: Regional Performance.
Target sheet: State Performance.
When selected: Filter to states in the clicked region.
This means clicking "South" on the regional bar chart immediately updates the state chart to show only Telangana, Tamil Nadu, Karnataka, Kerala, and Andhra Pradesh.

**Highlight action — hover on a category to highlight sub-categories:**
Source sheet: Category Profitability.
Target: All other sheets.
When hovered: highlights matching category rows without filtering away others.

**Reset filters:**
A "Reset all" button is created via a calculated field `""` placed on a sheet with all dimensions removed. Clicking it clears all dashboard filters by triggering a filter action that selects all members.

### Design decisions for filters

- All filters placed in a single panel at the top of the dashboard so their location is predictable
- Filter labels use sentence case ("All regions", not "ALL REGIONS")
- The record count badge updates in real time to confirm how many orders match the current selection
- An "active filter" strip below the filter panel shows badges for each currently-active filter in green

---

## 7. Key Business Insights

### Revenue and growth

- Total sales: **₹21.70 Cr** across 4,200 orders over 2 years
- Year-on-year growth: **+4.3%** (₹10.62 Cr in 2024 → ₹11.08 Cr in 2025)
- Margin held broadly stable: 15.5% in 2024, 15.2% in 2025 — growth came with a slight margin cost
- Highest month: August 2025 at ₹1.09 Cr
- Lowest month: August 2024 at ₹63 lakh — same month, consecutive years, opposite extremes

### Regional performance

| Region | Sales | Margin | Return rate | Order count |
|---|---|---|---|---|
| South | ₹6.47 Cr | 15.4% | 4.5% | 1,210 |
| North | ₹5.46 Cr | 15.2% | 4.5% | 1,056 |
| West | ₹4.89 Cr | 15.1% | 4.3% | 1,037 |
| East | ₹4.89 Cr | **15.6%** | 4.9% | 897 |

South leads on revenue. East leads on margin despite the lowest order count — an underserved region with quality demand.

### Category profitability

| Category | Sales share | Profit share | Margin |
|---|---|---|---|
| Technology | 70.9% | **84.2%** | 18.2% |
| Furniture | 23.8% | 10.7% | 6.9% |
| Office Supplies | 5.3% | 5.1% | 14.9% |

Technology is the profit engine. Furniture generates nearly a quarter of revenue but only a tenth of profit, with a return rate of 7.7% — more than double Technology's 3.0%.

### Discount impact on margin

| Discount tier | Orders | Avg margin |
|---|---|---|
| No discount (0%) | 1,024 | 19.0% |
| Low (1–10%) | 979 | 17.0% |
| Medium (11–20%) | 1,047 | 13.6% |
| High (21–30%) | 1,086 | 8.0% |
| Very high (31–35%) | 64 | **−4.4%** (loss-making) |

Every 10 percentage points of additional discount costs approximately 3–4 pp of margin. The highest tier is loss-making on average.

### Segment return risk

| Segment | Orders | Return rate | vs estate avg |
|---|---|---|---|
| Home Office | 1,446 | **5.67%** | +1.1 pp above average |
| Corporate | 1,399 | 4.00% | Below average |
| Consumer | 1,355 | 3.91% | Below average |

Home Office customers return 46% more orders proportionally than the estate average. Home Office + Furniture specifically: **9.1% return rate** — the highest combination in the dataset.

### Shipping performance

| Ship mode | Avg delivery | Return rate | % delayed (6+ days) |
|---|---|---|---|
| Same Day | 0.4 days | 2.5% | 0% |
| First Class | 1.8 days | 5.1% | <1% |
| Second Class | 2.7 days | 4.6% | 1% |
| Standard Class | 4.7 days | 4.6% | **27%** |

Standard Class handles 58% of all orders but delivers 27% of those outside the 3–5 day window. Same Day has the lowest return rate of any mode (2.5%).

---

## 8. Dashboard Story Summary

The business is growing — revenue increased 4.3% year-on-year — but the growth is concentrated and carries hidden strain.

**The South region and Technology category are doing the heavy lifting.** South generates 29.8% of total sales and holds four of the top five revenue states. Technology generates 84.2% of total profit at 18.2% margin. Together, these two concentrations mean the business is exceptionally dependent on a single geography and a single product category. This is efficient but fragile.

**Furniture is the most urgent strategic problem.** It consumes 23.8% of revenue resources but generates only 10.7% of profit at a 6.9% margin — the lowest in the estate. Its return rate of 7.7% is more than double Technology's and is worst among Home Office customers (9.1%). The combination of thin margin and high returns makes Furniture the most expensive category to operate per pound of profit generated.

**Two operational problems are quietly destroying value.** Standard Class shipping delivers 27% of its orders late, creating service failures and elevated return risk for 58% of all transactions. The 35% discount tier — 64 orders — is loss-making at −4.4% average margin, representing a deliberate or uncontrolled decision to sell below cost.

**The East region and Referral channel are underexploited.** East has the highest profit margin (15.6%) of any region but the fewest orders. Referral generates the highest revenue per order (₹54,111) of any channel at a strong 15.7% margin, yet handles only 9.3% of all orders. Both signal quality demand that the business has not yet scaled.

The dashboard converts these patterns into six prioritised actions: audit the 35% discount tier immediately, review Standard Class delivery performance within 30 days, rationalise the Furniture range within 60 days, evaluate the Paid channel's true ROI within 60 days, target East-region growth within 90 days, and investigate Home Office Furniture returns within 60 days.

---

## 9. Assumptions and Limitations

### Data assumptions

| Assumption | Detail |
|---|---|
| `sales` is post-discount revenue | The field reflects what the customer paid, not list price. `discount` captures the reduction applied. |
| `delivery_days` = `ship_date − order_date` | Verified: the two are identical for all 4,200 records. No imputation needed. |
| Negative profit = loss-making order | 288 orders have negative profit. These are valid business records (heavily discounted orders), not data errors. |
| Rajasthan is split across North and West | This is a district-level split confirmed in the data. Always filter by region + state together to avoid double-counting. |
| `SUM(sales)` = total revenue | No revenue exclusions applied. All order types are included in totals. |

### Dashboard limitations

**Missing cost-of-acquisition data.** The Paid channel's gross margin of 15.1% does not deduct media spend. Without it, true channel profitability cannot be compared. This is the most significant analytical gap in the current dataset.

**No return reason codes.** The `return_flag` is binary (0/1) — it identifies that a return occurred but not why. Root-cause analysis (size mismatch, transit damage, description mismatch) requires a separate data collection effort.

**4-month dataset coverage gaps.** While the period is 24 months, only 4 calendar months repeat across both years (January, February, March, April in each year are well-populated; holiday effects cannot be reliably isolated). Seasonal decomposition requires a minimum of 36–48 months.

**Customer rating excluded.** `customer_rating` has 32 missing values and a near-zero correlation with sales (r = −0.03). It is excluded from all dashboard metrics. It may be worth modelling as a dependent variable (what drives satisfaction?) rather than a predictor.

**Association is not causation.** The relationships shown in this dashboard — between discount and margin, between delivery speed and return rate, between region and profitability — are correlations observed in the data. They do not prove causal relationships. Pilot tests at a subset of locations are required before committing estate-wide changes based on dashboard findings.

**Model will become stale.** Coefficients and benchmarks in this dashboard reflect January 2024 to December 2025 trading conditions. The dashboard should be refreshed with at minimum 12 months of additional data annually.

---

## Repository Structure

```
part4_tableau_dashboard/
│
├── data/
│   ├── dashboard_sales_data.xlsx          # Full dataset — 4,200 orders, 20 columns
│   └── dashboard_sales_data_200kb.xlsx    # Reduced file (1,750 rows, ~200 KB)
│
├── tableau/
│   └── executive_dashboard.twbx           # Packaged workbook with embedded data
│
├── outputs/
│   ├── dashboard_story.md                 # Leadership narrative — 8 sections
│   ├── business_insights.md               # Key findings in plain language
│   ├── calculated_fields.md               # All 8 calculated fields with formulas
│   ├── tableau_sheets_guide.md            # Step-by-step build instructions per sheet
│   └── chart_selection_justification.md   # Design decisions for each chart
│
├── screenshots/
│   ├── full_dashboard.png                 # Complete dashboard view
│   ├── sales_trend_view.png               # Chart 1 close-up
│   ├── regional_performance_view.png      # Chart 2 close-up
│   ├── category_profitability_view.png    # Chart 3 close-up
│   └── filter_interaction_view.png        # Filter interaction evidence
│
└── README.md                              # This file
```

---

## Quick Reference

```
Dataset period     : January 2024 – December 2025
Total orders       : 4,200
Unique customers   : 4,096
Regions            : 4 (East, North, South, West)
States             : 20
Categories         : 3 (Furniture, Office Supplies, Technology)
Sub-categories     : 13
Total sales        : ₹21.70 Cr
Total profit       : ₹3.33 Cr
Overall margin     : 15.3%
Return rate        : 4.55%
Avg order value    : ₹51,671
Avg delivery       : 3.6 days
Calculated fields  : 8
Dashboard sheets   : 9 (7 charts + KPI tiles + dashboard canvas)
Filters            : 6 (Year, Region, Category, Segment, Ship Mode, Channel)
```

---

*Prepared by: Business Analysis Team*
*Project: `part4_tableau_dashboard`*
*Data source: `dashboard_sales_data.xlsx`*
*Last updated: June 2026*
More than 15% of orders fall into the Slow or Very Slow buckets. These are customers who waited 6 or more days for delivery — a meaningful service quality risk, particularly for Standard Class customers who may have expected 3–5 days. Tracking the Shipping Delay Bucket by region and ship mode reveals whether slow deliveries are concentrated in specific geographies or shipping methods.

### How to use in the dashboard

Use `Shipping Delay Bucket` as a colour dimension on a bar chart of order volume. Apply a custom colour order: Same Day = dark green, Fast = light green, Standard = grey, Slow = amber, Very Slow = red. This gives a traffic-light view of delivery performance at a glance.

> **Note for Tableau:** This field produces a **string/dimension**, not a measure. To control the sort order in charts, right-click the field in the view → **Sort** → Manual → drag to: Same Day, Fast, Standard, Slow, Very Slow.

---

## Calculated Field 6 — Discount Tier

### Formula

```
IF [discount] = 0 THEN "No Discount (0%)"
ELSEIF [discount] <= 0.10 THEN "Low (1–10%)"
ELSEIF [discount] <= 0.20 THEN "Medium (11–20%)"
ELSEIF [discount] <= 0.30 THEN "High (21–30%)"
ELSE "Very High (31–35%)"
END
```

### What it calculates

`discount` is stored as a decimal (0.25 = 25%) and takes only 8 discrete values (0, 0.05, 0.10, 0.15, 0.20, 0.25, 0.30, 0.35). Discount Tier converts this into readable percentage bands that align with business language and make profitability comparisons across discount levels immediately clear.

### Results in this dataset

| Tier | Orders | Avg Profit Margin |
|---|---|---|
| No Discount (0%) | 1,024 | 19.0% |
| Low (1–10%) | 979 | 15.8% |
| Medium (11–20%) | 1,047 | 12.5% |
| High (21–30%) | 1,086 | 6.8% |
| Very High (31–35%) | 64 | −4.4% |

### Business meaning

Each tier up in discount costs approximately 3–4 percentage points of margin. The Very High tier (31–35%) is loss-making on average. This field makes it easy to answer "what discount level is the business applying most often, and what does it cost in margin?" without requiring any manual data manipulation.

---

## Calculated Field 7 — Net Revenue After Discount

### Formula

```
[sales] * (1 - [discount])
```

### What it calculates

The `sales` field already reflects the post-discount revenue (it is the amount the customer actually paid). This field makes the discount impact explicit by showing what revenue would have been at full price versus the recorded discounted amount. Use it to calculate discount value given away:

```
Discount Value Given Away = [sales] / (1 - [discount]) - [sales]
```

Or simply use `[sales]` directly for all revenue analysis, since it already represents actual received revenue. This field is most useful for "what-if" analysis — how much more revenue would the business have earned if no discounts were applied.

---

## Calculated Field 8 — Customer Type (New vs Repeat)

### Formula

```
IF { FIXED [customer_id] : COUNTD([order_id]) } > 1
THEN "Repeat Customer"
ELSE "New Customer"
END
```

### What it calculates

This uses a **Level of Detail (LOD)** expression — one of Tableau's most powerful features. The `{ FIXED [customer_id] : COUNTD([order_id]) }` part calculates, for each customer, how many distinct orders they have placed across the entire dataset — regardless of what filters or groupings are applied in the current view.

If a customer has placed more than one order, they are labelled "Repeat Customer". If they have placed exactly one order, they are "New Customer".

### Why LOD is needed here

Without the `FIXED` expression, Tableau would count orders only within the current view's context. For example, if the view is filtered to "Region = South", a customer who ordered once in the South and once in the North would appear as a "New Customer" in the South view — incorrect. The `FIXED` expression ignores the view filter and always looks at the customer's full order history.

### Results in this dataset

| Type | Customers |
|---|---|
| New Customer | 3,992 |
| Repeat Customer | 104 (2.5%) |

### Business meaning

Only 2.5% of customers have placed more than one order in the two-year dataset. This is either a very low retention rate (a concern) or a reflection of high-value one-time purchases (less concerning for high-ticket Technology products). Tracking repeat customers by segment and channel helps identify which acquisition sources produce more loyal customers.

---

## Summary Table — All Calculated Fields

| Field name | Output type | Formula type | Primary use |
|---|---|---|---|
| Profit Margin (%) | Measure (%) | Aggregate | Margin analysis by category / region |
| Cost | Measure (₹) | Aggregate | Cost structure breakdown |
| Average Order Value | Measure (₹) | Aggregate | Channel and segment quality |
| Return Rate (%) | Measure (%) | Aggregate | Returns by category / region |
| Shipping Delay Bucket | Dimension (string) | Row-level IF/THEN | Delivery performance grouping |
| Discount Tier | Dimension (string) | Row-level IF/THEN | Discount impact analysis |
| Net Revenue After Discount | Measure (₹) | Row-level | What-if full-price analysis |
| Customer Type | Dimension (string) | LOD expression | New vs repeat customer split |

---

## Common Errors and Fixes

| Error message | Likely cause | Fix |
|---|---|---|
| "Cannot mix aggregate and non-aggregate" | Mixed `[sales]` and `SUM([sales])` in the same formula | Use `SUM()` consistently or neither — not both |
| "Expected type Number, found String" | Used a text field where a number is expected | Check that `[discount]` is a number, not stored as text |
| Shipping Delay Bucket sorts alphabetically | Tableau sorts strings A→Z by default | Right-click field in view → Sort → Manual → set custom order |
| Return Rate shows 0% for some segments | `return_flag` may be read as a dimension | Right-click `return_flag` in Data pane → Change to Measure → Sum |
| LOD formula shows "Unexpected FIXED" | Curly braces need to wrap the full expression | Ensure formula is: `{ FIXED [customer_id] : COUNTD([order_id]) }` with matching braces |

---

*Part of: `part4_tableau_dashboard` | Data source: `dashboard_sales_data.xlsx` | Calculated fields: 8 | June 2026*
