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
