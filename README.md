# Car Resale Data Pipeline (Medallion Architecture)

An ETL pipeline that turns messy, inconsistently-formatted car resale listings into a clean, analysis-ready dataset, following the Bronze/Silver/Gold medallion pattern, then feeds a Looker Studio dashboard.

Built as a hands-on data engineering task for **PwC's Jump Start Your Career programme** (Data Engineering track).

## Problem

Raw car resale data has currency symbols and text baked into numeric fields ("₹ 5.45 Lakh", "40,000 Kms", "83.1bhp"), inconsistent category spellings, and duplicate listings — none of which is usable directly for price comparison, depreciation analysis, or market trend reporting.

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

## Fixes applied for this repo

- **Made the notebook runnable outside Colab.** The original passed data between the Bronze/Silver/Gold sections using Colab's `files.upload()` / `files.download()` — meant the notebook couldn't run at all outside Colab (`google.colab` doesn't exist elsewhere) and required manually clicking through upload dialogs even inside it. Replaced with direct in-notebook DataFrame handoffs plus local reads/writes under `data/`, so `Run All` now reproduces the whole pipeline unattended.
- **`cleaned_resale_price` used `eval()`** on data-derived text as the fallback for plain numeric strings. Replaced with `float()`, which does the same job without evaluating arbitrary expressions.
- **`extract_mileage()` could crash on a value with no unit suffix** (`'kmpl' in unit` when `unit` is `None`). Added a guard. This doesn't currently trigger on this dataset — missing mileage values are already imputed with a modal string that includes a unit before this function runs — but it's now handled either way.
- **Insurance category had a trailing-space duplicate.** The `'2' -> 'Comprehensive '` mapping (note the trailing space) would have created a second, near-identical `'Comprehensive '` category alongside the existing `'Comprehensive'` entries instead of merging into it. Fixed the mapping.
- Dropped an unused `import requests`.

## Known limitations (not changed)

- **`norm_full()` assumes make and model are always exactly one word each** (`"2017 Maruti Baleno 1.2 Alpha"` → make=Maruti, model=Baleno). Multi-word makes like "Land Rover" or "Aston Martin" would be split incorrectly (e.g. make="Land", model="Rover"). A proper fix needs a make/model reference table rather than positional word-splitting - out of scope here.
- **Outlier handling only examined the `seats` column.** The notebook's outlier-handling step computes IQR bounds for `seats` (which turned out equal to the median, so nothing to trim) and concludes no outlier handling is needed — but doesn't check `resale_price`, `kms_driven`, or the other numeric columns, which are more likely to actually contain outliers in a resale dataset. Left as-is rather than adding new analysis that wasn't part of the original work.

## Files

```
data/
  car_resale_prices.csv   raw input
  car_bronze_layer.csv    Bronze layer output
  car_silver_layer.csv    Silver layer output
  car_gold_layer.csv      Gold layer output (feeds the dashboard)
car_resale_pipeline.ipynb
```

## Running it

```
pip install pandas matplotlib seaborn
jupyter notebook car_resale_pipeline.ipynb
```
Run all cells - it reads `data/car_resale_prices.csv` and writes the Bronze/Silver/Gold CSVs into `data/` as it goes.
