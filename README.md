# Airline Load Factor and Airfare Analysis

**Type:** Team project
**Contributors:** Carter Ward, Boyd Emmons, bodysnatcha
**Course:** Big Data course project
**Completed:** 04/13/2026

## Purpose

We built an end-to-end big-data pipeline on real, messy, multi-source U.S. airline industry data — not a toy dataset. We combine five government/industry sources covering competition, pricing, delays, fuel costs, and load factor for Q1 2025, scoped to airports in California, Georgia, and Texas, to model what drives average airfare. The project demonstrates the full lifecycle: collection, cleaning, joining, feature engineering, modeling, and honest evaluation.

## Problem and Approach

We set out to test whether higher **load factor** (percent of seats filled) is associated with higher **average airfare** — the intuition being that fuller planes give airlines more pricing power. `avg_fare` is our target variable over a route-quarter feature set. We ran three complementary methods on the same data (`Model.py`):

- **Correlation Analysis** — a Pearson correlation matrix across load factor, competition, delay, fuel, and fare variables.
- **Linear Regression** — an interpretable model over a fixed 19-feature set, 80/20 train/test split (`random_state=42`).
- **PCA Regression** — the same features standardized and PCA-reduced (95% variance retained) before regression, to test whether dimensionality reduction helps or hurts.

## Structure and Methodologies

- **`Data.py`** — preprocessing and feature engineering: defines 26 target airports (11 CA, 4 GA, 11 TX), reads and filters the five raw sources to Q1 2025, writes cleaned CSVs to `Cleaned_Data/`, then merges everything into a route-quarter `Analysis_Table.csv`.
- **`Model.py`** — feature selection, the three modeling techniques, evaluation (RMSE/MAPE/R²/SNR/accuracy), and all plots/reports in `outputs/`.
- **`Main.py`** — entry point; runs the pipeline in order: preprocess → build analysis table → run models → evaluate and save outputs.
- **Libraries**: `pandas`/`numpy` for data wrangling, `scikit-learn` for `LinearRegression`, `PCA`, `StandardScaler`, `train_test_split`, and metrics; `matplotlib` (headless `Agg`) for charts.
- **Pipeline stages**: raw CSVs → cleaned/filtered per-source CSVs → merged `Analysis_Table.csv` (2,850 route-quarter rows, keyed on `ORIGIN`/`DEST`/`YEAR`/`QUARTER`) → model frame with lag/rolling load-factor features and a fixed 19-column set → trained models, predictions, and evaluation outputs.

## Process

1. **Collect** five Q1 2025 sources: flight-level competition data (~93 MB), DB1B ticket data by state (~104 MB), monthly airport delay-cause data, daily jet fuel prices, and T-100 load factor segment data.
2. **Clean** large files in 100,000-row chunks, filtering to Q1 2025 and the 26 target airports; coerce/null-clean numeric columns; write to `Cleaned_Data/` (~3.7M rows total).
3. **Engineer features**: aggregate each source to route-quarter grain, merge on join keys, compute `load_factor`, delay/cancellation shares, and an `is_saturated` flag.
4. **Prep for modeling**: add lag/rolling `load_factor`, median-impute gaps, lock in the 19-feature set with `avg_fare` as target.
5. **Model**: run correlation analysis, Linear Regression, and PCA Regression (10 components for 95% variance).
6. **Evaluate and visualize**: compute RMSE/MAPE/R²/SNR, generate correlation heatmap, scatter plots, model comparison and diagnostics dashboards.

## Outcome

Linear Regression outperformed PCA Regression on every metric: RMSE $65.04 vs $68.14, MAPE 21.59% vs 23.35%, R² 0.388 vs 0.328, accuracy 78.41% vs 76.65% — PCA compression didn't help. Our original hypothesis was not well supported: `load_factor` correlates with `avg_fare` at only 0.074, while `market_distance` (0.553) and `is_saturated` (0.219) were the strongest correlates. The largest regression coefficients belonged to delay-share features (e.g., `route_weather_delay_share` +4437.72), far outweighing load factor (−7.15). PCA needed 10 of 19 components to reach 95% variance, showing the feature set isn't very redundant. A single-quarter scope meant `avg_fuel_price` had zero variance, producing `NaN` correlations — a real data limitation we caught by inspecting outputs directly. On the 570-row test set, mean predicted fare was within about $6 of actual ($275.26 vs $269.43), with median absolute error around $32. Working across five differently-shaped datasets sharpened our skills in chunked reading, join-key/aggregation choices, and data cleaning — and taught us to trust the evidence over the hypothesis rather than cherry-pick a confirming result.

## How to Run

```bash
pip install -r requirements.txt
python Main.py
```
