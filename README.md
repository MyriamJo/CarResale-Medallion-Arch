# Car Resale Data Pipeline (Medallion Architecture)

An ETL pipeline that turns messy, inconsistently-formatted car resale listings into a clean, analysis-ready dataset, following the Bronze/Silver/Gold medallion pattern, then feeds a Looker Studio dashboard.

Built as a hands-on data engineering task for **PwC ETIC's Jump Start Your Career programme** (Data Engineering track).

## Problem

Raw car resale data has currency symbols and text baked into numeric fields, inconsistent category spellings, and duplicate listings, none of which is usable directly for price comparison, depreciation analysis, or market trend reporting.

## Pipeline

**Bronze layer** - raw listings ingested as-is; the only change is promoting the source file's row index into a proper DataFrame index.

**Silver layer** - cleaning and standardization:
- Drop fully-empty rows and exact-duplicate rows (~1.2% of rows).
- Standardize column names (lowercase, underscores).
- Impute missing values - median for numeric columns, mode for categorical columns (all under 5% missing), with a separate `Unknown` category for `insurance` instead of guessing.
- Parse `full_name` into `year` / `make` / `model` / `variant`.
- Strip currency symbols and unit text from `resale_price` (→ INR), `engine_capacity` (→ cc), `kms_driven`, `max_power` (→ bhp), and `mileage` (→ kmpl, converting km/kg to kmpl where needed).
- Standardize `transmission_type` and `owner_type` spelling/casing.

**Gold layer** - feature engineering and final shape:
- `vehicle_age` = current year − registered year.
- `price_per_km` = resale price / kms driven.
- `duplicate_record_flag` - flags listings that share `full_name`, `year`, and `kms_driven` (a softer duplicate signal than the Silver layer's exact-row check — catches likely re-listings that differ in a minor field).
- Columns reordered and renamed to the target Gold schema (`engine_capacity_cc`, `max_power_bhp`, `mileage_kmpl`, `vehicle_id`, etc.)

**Dashboard** - the Gold CSV feeds a [Looker Studio dashboard](https://lookerstudio.google.com/reporting/bbf7122e-257c-4adc-870d-c387e0712271) (built separately in Looker Studio, not from this notebook) with market composition and pricing-by-age/ownership views.
