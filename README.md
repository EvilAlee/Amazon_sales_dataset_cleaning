# Amazon Products Sales Data — Cleaning Report

**Dataset:** amazon_products_sales_data_uncleaned.csv  
**Source:** Kaggle  
**Domain:** E-commerce / Sales & Marketing  
**Problem:**  This dataset contains scraped Amazon product listings including product titles, ratings, review counts, purchase volume, pricing, seller information, delivery details, and product metadata. The raw dataset contained numerous missing values, inconsistent formatting, mixed data types, and web-scraping artifacts requiring extensive cleaning before analysis.


---

## Dataset Overview

| Property | Value |
|----------|-------|
| Rows (raw) | 42,675 |
| Rows (clean) | 37,989 |
| Columns (raw) | 16 |
| Columns (clean) | 15 |
| Rows dropped | 4,686 |

All 16 columns were originally stored as `str`. After cleaning: 4 `float64`, 1 `int64`, 1 `datetime64`, 9 `str`.

---

## Phase 1 — Exploration Findings

Ran full structural checks: `head()`, `info()`, `describe()`, `isnull().sum()`, `duplicated()`, `value_counts()` on all categoricals, and a null heatmap.

**8 columns had missing values:**

| Column | Nulls | % Missing |
|--------|-------|-----------|
| sustainability_badges | 39,267 | 92% |
| buy_box_availability | 14,653 | 34% |
| delivery_details | 11,720 | 27% |
| current/discounted_price | 11,749 | 27% |
| bought_in_last_month | 3,217 | 7.5% |
| product_url | 2,069 | 4.8% |
| rating | 1,024 | 2.4% |
| number_of_reviews | 1,024 | 2.4% |

**Additional issues found:**
- All numeric columns stored as strings with currency symbols (`$`), commas, and suffixes (`out of 5 stars`, `+ bought in past month`)
- `is_couponed` contained both percentage and dollar-amount discount values mixed in one column
- `is_best_seller` contained percentage discount values alongside badge labels
- `price_on_variant` was the most complex column — contained prices, coupon percentages, and "nan" strings mixed together
- `bought_in_last_month` used K-notation (`6K+`, `100K+`) requiring custom parsing
- No duplicate rows found

---

## Phase 2 — Cleaning Steps Applied

### Step 1 — Drop rows with missing `product_url`
Rows without a source URL cannot be traced or verified. Dropped 2,069 rows.

> If priority were data visualization rather than accuracy, these rows would have been kept.

### Step 2 — Convert `collected_at` to datetime
No nulls present. Converted directly using `pd.to_datetime()`.

### Step 3 — Clean `listed_price`
- Replaced `"No Discount"` string with `NaN`
- Stripped `$` and `,` using regex
- Cast to `float64`

### Step 4 — Fill `sustainability_badges`
92% missing. Since absence of a badge is itself meaningful, filled nulls with `"Unknown"` rather than dropping.

### Step 5 — Clean `rating`
- Stripped `" out of 5 stars"` suffix using `str.replace()`
- Cast to `float64`
- Checked for outliers using `describe()` — none found (range: 1.0–5.0)
- Filled 1,024 nulls using **group median** by `is_sponsored` and `sustainability_badges`

### Step 6 — Clean `number_of_reviews`
- Removed comma separators using `str.replace()`
- Cast to `float64`
- Chose **median over mean** for null filling — mean was 2,891 vs median 336, indicating heavy right skew from viral products
- Filled nulls using **group median** by `rating` and `is_sponsored`
- Final cast to `int64`

### Step 7 — Clean `bought_in_last_month`
This column required the most custom work due to K-notation:
- Extracted numeric prefix using `str.extract(r'(^\d+K?)')`
- Applied custom `fixer()` function to convert `"6K"` → `6000.0` and `"300"` → `300.0`
- Filled nulls in two passes:
  1. **Group median** by `number_of_reviews` — buyers correlate with review count
  2. **Global median fallback** for 205 groups that were entirely null (no non-null values existed to derive a median from)

### Step 8 — Feature engineering for price columns
Extracted discount information into temporary helper columns:

| Column | Source | Logic |
|--------|--------|-------|
| `cp%` | `is_couponed` | Coupon percentage (`Save 15%`) |
| `cp$` | `is_couponed` | Coupon dollar amount (`Save $16.00`) |
| `bg%` | `is_best_seller` | Best seller percentage discount |
|`price$`| `listed_price`  | Leftover price value not found in current price |

Cross-verified `bg%` against `cp%` — found 75 rows where both existed but percentages were non-stacking, so `bg%` was ignored for price imputation.

### Step 9 — Price imputation using business logic
Rather than filling prices with statistical methods (mean/median), missing prices were recovered mathematically where discount information was available:

**For `listed_price` nulls where `cp%` and `current/discounted_price` were known:**
```
listed_price = current/discounted_price / (1 - cp%)
```

**For `listed_price` nulls where `cp$` and `current/discounted_price` were known:**
```
listed_price = current/discounted_price + cp$
```

**For `current/discounted_price` nulls where `price_on_variant` contained a clean price:**
- Created mask using regex `price:\s\$\d+\.\d+$` to isolate rows with exact prices (not coupons or percentages)
- Found 8,852 such rows — extracted price and filled both `current/discounted_price` and `listed_price`

**For remaining `listed_price` nulls where `current/discounted_price` existed:**
- 16,042 rows where listed price was null but current price was available
- Reasoned that if no discount exists, the listed price equals the current price
- Confirmed via mask analysis that no recoverable data was being lost

### Step 10 — Drop unrecoverable `current/discounted_price` nulls
After all imputation passes, 2,617 rows still had no recoverable current price from any source. Dropped these rows.

> Statistical filling (mean/median) was rejected here — a meaningless price value is worse than no row at all for any downstream analysis or ML model.

After this drop, all remaining `listed_price` nulls were also eliminated (they belonged to the same rows).

### Step 11 — `buy_box_availability` and `delivery_details`
Heatmap analysis revealed that null patterns in these two columns were correlated — rows with null `buy_box_availability` also had null `delivery_details`. This indicated out-of-stock products where neither field had data.

- Created co-relation mask: both columns null simultaneously (8,905 rows)
- Filled `buy_box_availability` nulls → `"Out of stock"`
- Filled co-related `delivery_details` nulls → `"Out of stock"`
- Filled remaining `delivery_details` nulls → `"Unknown"`

### Step 12 — Cleanup
- Dropped 5 helper columns: `price_on_variant`, `cp%`, `cp$`, `bg%`, `price$`
- Restored `bought_in_last_month` to original readable format using `unfixer()`:
  - Values ≥ 1000 → `"6K+ bought in past month"`
  - Values < 1000 → `"300+ bought in past month"`

---

## Phase 3 — Validation

Post-cleaning `isnull().sum()` confirmed **zero nulls across all 15 columns**.

Final heatmap showed a fully black grid — no white lines.

**Final dtypes:**

| Type | Columns |
|------|---------|
| `float64` | rating, bought_in_last_month, current/discounted_price, listed_price |
| `int64` | number_of_reviews |
| `datetime64[us]` | collected_at |
| `str` | all remaining 9 columns |

---

## Key Decisions

| Decision | Reasoning |
|----------|-----------|
| Drop `product_url` nulls | Row has no traceable source |
| Fill `sustainability_badges` with `"Unknown"` | Absence of badge is meaningful, not missing |
| Group median for `rating` and `reviews` | Smarter than global fill — accounts for sponsored vs organic patterns |
| Two-pass fill for `bought_in_last_month` | Some review-count groups had no non-null values at all |
| Mathematical price recovery | More accurate than statistical imputation for financial data |
| Drop remaining `current/discounted_price` nulls | No source to recover from — accuracy over row count |
| Co-relation fill for `buy_box_availability` / `delivery_details` | Pattern observed from heatmap, confirmed by mask |

---

## Techniques Used  

- Missing value analysis
- Regular Expressions (Regex)
- Feature Engineering
- Boolean Masking
- `groupby().transform()`
- Median and grouped mean imputation
- Relationship-based imputation
- Datetime conversion
- Type conversion
- Exploratory Data Analysis (EDA)  

---
As I mentioned in Notebook this project prioritizes **data accuracy over retaining every row**. Instead of using simple mean or median imputation everywhere, missing values were reconstructed whenever logical relationships existed between columns (prices, coupons, review counts, stock availability, etc.). Rows with unrecoverable critical pricing information were removed to preserve dataset quality.


