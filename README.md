# Part 1 — Data Cleaning Project

**Course Assignment | Business Data Analysis**

---

## Business Scenario

A retail analytics team received an export of order data from an e-commerce platform covering 2024–2025. Before any reporting can happen, the dataset needs a full audit and cleanup pass — missing values, inconsistent formats, duplicate records, invalid discount entries, and broken calculations are all mixed in. This project documents that process from raw data to analysis-ready output.

---

## Dataset Description

| Field | Description |
|---|---|
| order_id | Unique order identifier (some duplicated) |
| order_date / ship_date | Order and shipping dates — mixed formats |
| customer_id / customer_name | Customer reference data |
| segment | Customer segment (Consumer, Corporate, Small Business, Home Office) |
| region | North / South / East / West (some missing) |
| state / city | Location details |
| category / sub_category | Product hierarchy |
| ship_mode | Shipping method (some missing) |
| quantity / unit_price / discount | Order line details |
| sales / cost / profit | Financial figures |
| payment_status | Paid / Pending / Failed / Refunded |
| order_status | Completed / Cancelled / Returned |

**Raw rows:** 932 | **After cleaning:** 912 (20 exact duplicates removed)

---

## Folder Structure

```
part1_data_cleaning/
├── data/
│   ├── raw_orders.xlsx          ← original file, untouched
│   └── cleaned_orders.xlsx      ← cleaned output with added columns
├── outputs/
│   ├── data_quality_report.xlsx ← 8-tab quality audit report
│   └── pivot_summary.xlsx       ← 9 pivot tables for business reporting
├── screenshots/
│   ├── raw_data_preview.png
│   ├── cleaned_data_preview.png
│   ├── pivot_summary_1.png
│   └── pivot_summary_2.png
├── cleaning_log.md              ← this analyst's working notes
└── README.md
```

---

## Tools Used

- **Python** (pandas, openpyxl, dateutil) — cleaning logic and Excel generation
- Operations mirror manual Excel work: TRIM, PROPER, UPPER, LOWER, IF, find-and-replace, remove duplicates, filters
- **LibreOffice** — formula recalculation verification

---

## Cleaning Steps

1. **Date standardization** — six different date formats identified and parsed to `YYYY-MM-DD`
2. **Text normalization** — trimmed whitespace, standardized capitalization (region, category, ship_mode, segment, etc.)
3. **Missing values** — region → "Unknown", ship_mode → "Unknown", discount → 0.0 (where valid)
4. **Exact duplicates removed** — 20 rows with identical key fields
5. **Conflicting duplicates flagged** — 31 order IDs with same ID but different data
6. **Discount cleaning** — converted "70%" to 0.70, flagged negatives and values above 50%
7. **Calculated columns added** — see below
8. **Data quality flags** — every row gets a flag; clean rows show "OK"

---

## Business Rules

| Rule | Applied As |
|---|---|
| Missing region | Filled "Unknown", flagged |
| Missing ship mode | Filled "Unknown", flagged |
| Missing discount | Treated as 0 if other fields valid; else flagged |
| Negative discount | Flagged Invalid, set to 0 in calculated_sales |
| Discount > 50% | Flagged Invalid, set to 0 in calculated_sales |
| Ship date before order date | Flagged, record retained |
| Exact duplicates | Removed (kept first) |
| Conflicting duplicate IDs | Flagged, retained for review |
| Cancelled / Failed orders | Excluded from completed-sales summaries |
| Refunded orders | Tracked separately in pivot |

---

## Calculated Columns Added

| Column | Formula Logic |
|---|---|
| `cleaned_discount` | Discount value cleaned; negatives and >50% set to 0 |
| `calculated_sales` | `quantity × unit_price × (1 − cleaned_discount)` |
| `calculated_profit` | `sales − cost` |
| `profit_margin` | `calculated_profit / sales × 100` |
| `shipping_delay_days` | `ship_date − order_date` in days |
| `order_month` | Month name from order_date |
| `order_year` | Year from order_date |
| `data_quality_flag` | Pipe-delimited list of all issues per row; "OK" if clean |

---

## Data Quality Findings

| Issue | Count |
|---|---|
| Missing region | 26 |
| Missing ship mode | 22 |
| Missing discount | 18 |
| Negative discounts | 16 |
| Discount > 50% | 15 |
| Exact duplicates removed | 20 |
| Conflicting duplicate order IDs | 31 |
| Ship date before order date | 13 |
| Sales calculation mismatch | 65 |
| Profit calculation mismatch | 41 |
| Failed payment records | ~58 |
| Cancelled orders | ~146 |
| Refunded orders | ~71 |

Full breakdown in `outputs/data_quality_report.xlsx`.

---

## Pivot Reports

| Sheet | Description |
|---|---|
| Sales by Region | Total sales and profit per region, sorted descending |
| Profit by Region | Profit and margin per region |
| Sales by Category | Furniture / Technology / Office Supplies breakdown |
| Sales by Sub Category | Detailed product line view, sorted by sales |
| Margin by Segment | Profit margin ranked by segment |
| Orders by Ship Mode | Delivery method distribution |
| Refunded by Region | Refunded order volume by geography |
| Cancelled by Region | Cancelled order volume by geography |
| Monthly Sales Trend | Month-by-month sales 2024–2025 |

All pivot reports cover only **Completed + Paid** orders (except refunded/cancelled sheets which show their respective populations).

---

## Key Insights

- Sales mismatch is a significant issue — 65 rows (~7%) have a difference between recorded and calculated sales, suggesting upstream data quality problems in the source system
- South and East regions handle the highest order volumes based on the completed data
- Standard Class is the most common shipping method
- Furniture tends to drive higher absolute sales, while Office Supplies has more order volume at lower per-order values
- The refunded order rate across regions appears fairly consistent — no single region stands out as having a disproportionate returns problem

---

## Assumptions

- Dates were parsed with `dayfirst=True` throughout — consistent with an Indian business dataset
- Discounts above 50% are treated as data entry errors, not valid promotional discounts
- For `calculated_sales`, invalid discounts are replaced with 0 rather than propagating the error
- "Completed" orders with Pending or Failed payment status are excluded from sales summaries — a completed delivery with failed payment is an accounts receivable issue, not a clean sale

---

## Limitations

- The sales mismatch in 65 rows cannot be resolved without checking the source transaction system
- Ambiguous date formats (e.g., `05/08/2024`) could be January 5th or August 5th — parsing was consistent but not guaranteed correct for every single row
- Conflicting duplicate order IDs (31 cases) need manual business review to determine the authoritative version
