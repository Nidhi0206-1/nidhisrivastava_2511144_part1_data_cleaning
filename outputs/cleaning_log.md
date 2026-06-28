# Data Cleaning Log — raw_orders.xlsx

**Dataset:** raw_orders.xlsx (Indian e-commerce order data)  
**Tool Used:** Python (pandas, openpyxl) — operations mirror what you'd do manually in Excel  
**Rows in:** 932 | **Rows out:** 912 (20 exact duplicates removed)

---

## Issues Found

### 1. Date Format Mess
The order_date and ship_date columns had at least six different formats mixed together across rows — `21 Jul 2024`, `08/31/2024`, `2024-05-24`, `28-11-2024`, `07 Feb 2024`, and `03/01/2024`. This created a secondary problem: some rows appeared to have ship dates before order dates, but in many cases it was just a DD/MM vs MM/DD parsing issue (e.g., `03/01/2024` could be January 3rd or March 1st depending on which side you're on). After carefully parsing with dayfirst=True, the genuinely problematic cases (where ship date was actually earlier) were flagged.

### 2. Negative Discounts
16 rows had negative discount values (like `-0.19`, `-0.23`, `-0.14`). These don't make sense as discounts — a negative discount would mean the customer paid *more* than the listed price, which isn't a valid business scenario. These were flagged as `Invalid Discount (Negative)`.

### 3. Discount > 50% (and % Format Issues)
Several rows had discounts listed as `70%`, `85%` — stored as text with a % sign. Others had decimal values like `0.55` or `0.65`. All of these were flagged as unusually high discounts. The % values were converted to decimals first (70% → 0.70), then flagged. A discount above 50% is almost certainly a data entry error for a standard retail business.

### 4. Missing Region (26 rows)
Region was blank for 26 records. Per business rules, these were filled with `Unknown` so the records aren't lost — they can still be counted in totals and investigated later.

### 5. Missing Ship Mode (22 rows)
Same approach as region. Filled with `Unknown`. These could be orders where the shipping method wasn't recorded at point of entry.

### 6. Missing Discount (18 rows)
Where discount was blank but all other sales fields (quantity, unit_price, sales, cost, profit) looked consistent, I treated the discount as 0. Where the missing discount caused a sales mismatch, both flags were applied.

### 7. Exact Duplicate Rows (20 removed)
20 rows were perfect copies — same order ID, same dates, same customer, same sales and profit figures. These look like accidental double-entries. Removed, keeping the first occurrence.

### 8. Conflicting Duplicate Order IDs (31 flagged)
These had the same order ID but different values — different sales figures, shipping details, or statuses. Examples: `ORD-2024-10124` appeared three times with different sales figures. Rather than deleting them blindly, I flagged them all with `Conflicting Duplicate (Flagged)` so a business user can review and decide which version is correct.

### 9. Sales Calculation Mismatch (65 rows)
`sales` should equal `quantity × unit_price × (1 − discount)`. In 65 cases, the recorded sales value didn't match this formula by more than ₹1. Some of these overlapped with the invalid discount cases (negative or >50% discounts produce obviously wrong calculated sales). These rows are flagged but the original recorded sales value is kept — I didn't overwrite it. The `calculated_sales` column shows what it *should* be.

### 10. Profit Mismatch (41 rows)
`profit` should equal `sales − cost`. In 41 rows, the difference between recorded profit and (sales − cost) exceeded ₹1. Flagged similarly — original values preserved, `calculated_profit` added as a reference column.

### 11. Capitalization & Extra Spaces
- `region`: values like `NORTH`, `north`, `  North `, `west` all standardized to `North`, `South`, `East`, `West`
- `category`: `OFFICE SUPPLIES`, `Office  Supplies`, `  Furniture ` etc. — all standardized
- `sub_category`: `PAPER`, `storage`, `  Accessories ` — standardized
- `ship_mode`: `STANDARD CLASS`, `Standard  Class`, `  First Class ` — standardized
- `segment`: `  Small Business `, `Small  Business`, `consumer` — standardized
- `customer_name`: `PRIYA MENON`, `Vikram  Iyer` (double space) — title cased and trimmed
- `payment_status` / `order_status`: `Paid ` (trailing space), `failed`, `CANCELLED`, `  Completed ` — all trimmed and title cased

### 12. Mixed / Invalid Order Status
Values like `completed`, `COMPLETED`, `  Completed `, `cancelled`, `CANCELLED` were all normalized to `Completed` / `Cancelled` / `Returned`. One row had `order_status = 'completed'` (lowercase) with `payment_status = 'Refunded'` which is contradictory — it's flagged accordingly.

---

## Cleaning Actions

| Action | Method |
|---|---|
| Date standardization | Parsed each date with format detection (dayfirst=True for DD-MM-YYYY formats), output as YYYY-MM-DD |
| Region fill | Blank → "Unknown" |
| Ship Mode fill | Blank → "Unknown" |
| Discount normalization | Strip %, convert to decimal, flag negatives and >0.5 |
| Text standardization | TRIM + PROPER equivalent — strip whitespace, title case, collapse double spaces |
| Exact duplicate removal | Removed 20 rows with identical key fields |
| Conflicting duplicate flagging | Kept all versions, added flag |
| Calculated columns | Added calculated_sales, calculated_profit, profit_margin, shipping_delay_days, order_month, order_year |

---

## Business Rules Applied

- **Missing region / ship mode** → filled `Unknown`, flagged
- **Missing discount** → treated as 0.0 if other fields are clean; otherwise also flagged as `Missing Discount`
- **Negative discount** → flagged `Invalid Discount (Negative)`, cleaned_discount set to 0 for the calculated_sales column
- **Discount > 50%** → flagged `Invalid Discount (>50%)`, cleaned_discount set to 0 for calculated_sales
- **Cancelled orders** → kept in dataset but excluded from completed-sales pivot summaries
- **Failed payments** → kept in dataset but excluded from completed-sales pivot summaries
- **Refunded orders** → tracked in a separate pivot (Refunded by Region)
- **Ship date before order date** → flagged, not deleted
- **Exact duplicates** → removed
- **Conflicting duplicate order IDs** → flagged, retained for business review

---

## Assumptions

- `dayfirst=True` parsing was used for ambiguous dates like `05/08/2024` — treated as August 5th, not May 8th. This is consistent with an Indian dataset.
- A discount above 50% is flagged as unusual, not automatically "wrong" — some legitimate promotions could go higher, but these looked like data entry errors in context.
- For the `calculated_sales` column, invalid discounts (negative or >50%) are replaced with 0% — meaning the calculation uses `quantity × unit_price` rather than applying a potentially wrong discount figure.
- Where `order_status = 'Completed'` and `payment_status = 'Pending'` or `Failed`, these are excluded from completed sales summaries. A completed order with a failed payment is a business problem, not a data cleaning problem, so both flags are preserved.

---

## Removed Records

| Reason | Count |
|---|---|
| Exact duplicate rows | 20 |
| **Total removed** | **20** |

---

## Flagged Records (kept, not removed)

| Flag | Count (approx) |
|---|---|
| Conflicting Duplicate Order IDs | 31 |
| Sales Mismatch | 65 |
| Profit Mismatch | 41 |
| Invalid Discount (Negative) | 16 |
| Invalid Discount (>50%) | 15 |
| Missing Discount | 18 |
| Missing Region | 26 |
| Missing Ship Mode | 22 |
| Ship Date Before Order Date | 13 |
| Failed Payment | ~58 |
| Cancelled Order | ~146 |
| Refunded Order | ~71 |

Many records carry multiple flags (e.g., a refunded order with a missing region).

---

## Limitations

- Date parsing is best-effort. For formats like `05/08/2024`, dayfirst=True was assumed consistently — if any dates in the dataset were actually MM/DD/YYYY, those would be parsed incorrectly. A business user should verify borderline cases.
- The 65 sales mismatches were flagged but not corrected. It's not clear whether the error is in the quantity, unit price, discount, or recorded sales value — the source system would need to be checked.
- Conflicting duplicate order IDs were flagged rather than resolved. A business user needs to confirm which version of each order is accurate.
- `profit_margin` is calculated as `(sales - cost) / sales × 100`. A few rows have near-zero sales values which could produce extreme margin percentages — these would typically be excluded from margin analysis.
