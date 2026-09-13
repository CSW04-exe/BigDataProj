# Airline Load Factor and Airfare Analysis

## 1. Purpose

This project exists to answer a question that comes up constantly in discussions of airline pricing: **does flying fuller planes actually mean flying more expensive planes?** "Load factor" — the percentage of available seats an airline actually sells — is one of the most closely watched numbers in the airline industry, and the intuitive story is that airlines under capacity pressure charge more. This repository builds an end-to-end data pipeline to test that intuition with real U.S. domestic flight data rather than taking it on faith.

Beyond the specific hypothesis, the project is also a personal exercise in handling data at a genuinely inconvenient scale: multi-hundred-megabyte government datasets that don't fit comfortably in memory, come from five unrelated sources, and have to be reconciled into a single clean table before any modeling can happen. It's a "big data" project less because any single file is enormous by industry standards, and more because the workflow — chunked ingestion, multi-source joins, feature engineering, and reproducible modeling — mirrors what a real analytics pipeline looks like.

## 2. Problem and approach

**The problem:** For a defined slice of the U.S. air travel market — California, Georgia, and Texas airports during Q1 2025 — determine whether route-level load factor is a meaningful predictor of average airfare, and figure out what else actually drives fares once load factor is on the table (competition, delays, fuel costs, distance).

**The approach** was to treat this like a small applied research project rather than a single script, splitting the work into six explicit stages that map directly onto the codebase:

1. **Data preprocessing** (`Data.py`) — ingest five independent government/industry datasets (flight-level operations, DB1B market ticket data, delay-cause statistics, fuel prices, and T-100 segment traffic), restrict everything to Q1 2025 and a fixed list of target airports (11 in California, 4 in Georgia, 11 in Texas), and write cleaned CSVs.
2. **Feature engineering** (`Data.py`) — aggregate the cleaned sources up to a shared "route-quarter" grain (origin-destination pair, year, quarter), computing load factor, competition counts, delay-cause shares, and a binary "saturated route" flag (load factor ≥ 80%).
3. **Feature selection** (`Model.py`) — add lag and rolling-average load-factor features, impute missing values, and lock in a fixed 19-feature modeling set.
4. **Expert techniques** (`Model.py`) — run three complementary analyses: Pearson correlation analysis, ordinary Linear Regression, and PCA Regression (standardize → reduce dimensionality → regress) to see whether dimensionality reduction changes the picture.
5. **Performance evaluation** (`Model.py`) — score both regression models with RMSE, MAPE, R², a signal-to-noise ratio, and a derived accuracy percentage.
6. **Visualization** (`Model.py`) — generate correlation heatmaps, actual-vs-predicted scatter plots, distribution histograms, a fare/load-factor time series, and a PCA variance curve.

`Main.py` orchestrates all six stages in order so the whole pipeline — from raw CSVs to final plots — runs with a single command.

## 3. Structure and methodologies

**Language and core libraries** (see `requirements.txt`):

- **Python 3.9+**
- **pandas** — chunked CSV ingestion (`chunksize=100_000` for the largest files), filtering, groupby aggregation, and merging five datasets into one analysis table
- **NumPy** — numeric operations for metrics (SNR, residuals) and array handling
- **scikit-learn** — `LinearRegression`, `PCA`, `StandardScaler`, `train_test_split`, and the regression metrics (`mean_squared_error`, `mean_absolute_percentage_error`, `r2_score`)
- **Matplotlib** (Agg backend, for headless/CI-safe rendering) — every visualization artifact the pipeline produces

**Data structures and design choices:**

- A **route-quarter table** (`outputs/Analysis_Table.csv`) is the central data structure the whole project revolves around — one row per (year, quarter, origin, destination) combination, built by successively merging DB1B fare/passenger data, T-100 capacity data, competition counts, fuel prices, and airport-level delay statistics, then averaging origin/destination delay features to the route level.
- **Chunked streaming reads** rather than a single `read_csv` call for the two largest raw files (the flight-level competition dataset and the state-level DB1B ticket files), which keeps memory bounded while filtering down to the relevant date range and airports before concatenation.
- A **fixed, explicit 19-feature set** (load factor plus its lag/rolling variants, market distance, fuel price, passenger volume, competition metrics, and six delay-share/rate features) is used for both Linear Regression and PCA Regression, so the two techniques are compared on equal footing.
- **PCA with a variance threshold** (`n_components=0.95`) rather than a fixed component count, so the model keeps only as many components as needed to explain 95% of variance (it ends up needing more than 10 of the ~19 original features to hit that bar).
- Modular separation of concerns across three files: `Data.py` (ingest + feature engineering), `Model.py` (modeling, evaluation, plotting), `Main.py` (pipeline orchestration) — each stage is independently callable and testable.

**Directory layout:**

```text
Data.py                  # Stage 1 (preprocessing) + Stage 2 (feature engineering)
Model.py                 # Stage 3 (feature selection) + Stage 4 (modeling)
                          # + Stage 5 (evaluation) + Stage 6 (visualization)
Main.py                  # Orchestrates the full pipeline in stage order
Project Datasets/        # Raw source data (Competition, DB1B, Delays, Fuel, T-100)
Cleaned_Data/            # Filtered, cleaned CSVs written by Stage 1
outputs/Analysis_Table.csv   # Final route-quarter feature table
outputs/modeling/        # Correlation matrix, overview panel, prediction summaries
outputs/evaluation/      # Metrics, comparison plots, diagnostics, conclusions
requirements.txt         # pandas, numpy, scikit-learn, matplotlib
```

## 4. Process

Reconstructing the build from how the code is structured, the project came together roughly in this order:

1. **Sourcing and scoping the data.** Five separate real-world datasets had to be tracked down — Bureau of Transportation Statistics-style flight operations data, DB1B market ticket data (split by state, since national files are too large to handle at once), airline delay-cause statistics, Gulf Coast jet fuel price history, and T-100 segment traffic data. Rather than trying to model the entire U.S. network, the scope was deliberately narrowed to a fixed set of airports across three states and a single quarter (Q1 2025), which made the problem tractable without abandoning realism.

2. **Building the preprocessing layer first.** `Data.py`'s `_load_*` helper functions each handle one dataset's quirks — different column names, different date formats, different filtering needs — before anything gets merged. The chunked reading pattern for the largest files (`chunksize=100_000`) suggests this had to be added after an initial attempt at a plain `read_csv` was too slow or memory-heavy for the raw file sizes (the cleaned competition and DB1B files alone are 43MB and 93MB after filtering down to three states).

3. **Designing the merge keys.** Getting five datasets with different natural grains (flight-level, ticket-level, airport-level, day-level, segment-level) onto a single `(YEAR, QUARTER, ORIGIN, DEST)` key was the central engineering challenge — visible in how carefully `_agg_competition`, `_agg_delay`, `_agg_db1b`, `_agg_t100`, and `_agg_fuel` each roll their source up to a comparable grain before `build_analysis_table` joins them together, including the origin/destination delay features that get computed twice (once per airport role) and then averaged to a route-level number.

4. **Feature engineering on top of the merged table.** Once the base table existed, derived features were layered on: load factor as a ratio, a saturation flag at an 80% threshold, and — inside `Model.py`'s `_build_model_frame` — lag and rolling-average load-factor features computed per route after sorting by time, with a fallback to the current value when no prior quarter exists (since Q1 2025 is the only quarter in scope, this fallback is exercised for effectively every row).

5. **Modeling, iterated as a pair.** Linear Regression and PCA Regression were built side by side rather than sequentially — the code trains both on an identical 80/20 split (`random_state=42` for reproducibility) and reuses the same test set for both, which points to the two models being designed for direct, fair comparison from the outset rather than PCA being bolted on afterward.

6. **Evaluation and visualization as a final, separate pass.** `Model.py`'s evaluation functions (`_plot_model_comparison`, `_plot_actual_vs_predicted`, `_plot_diagnostics`, `_save_feature_importance`, `_save_conclusions`) are cleanly separated from the training code and write everything to `outputs/evaluation/` as both plots and human-readable `.txt` tables — a sign that generating an inspectable, reviewable output was treated as a first-class goal of the project, not an afterthought bolted on for a grade.

## 5. Outcome

Running the full pipeline (`python Main.py`) on the Q1 2025 California/Georgia/Texas dataset produces committed, reproducible results in `outputs/evaluation/`:

| Model | RMSE | MAPE | R² | SNR | Accuracy (1 − MAPE) |
|---|---|---|---|---|---|
| **Linear Regression** | $65.04 | 0.216 | 0.388 | 17.16 | **78.41%** |
| PCA Regression | $68.14 | 0.234 | 0.328 | 15.64 | 76.65% |

Linear Regression came out ahead on every metric — a realistic result, since PCA Regression trades some predictive accuracy for dimensionality reduction, and in this case the original 19-feature space (needing over 10 principal components to retain 95% of variance) didn't compress cleanly enough to beat the plain model.

On the **original hypothesis**, the answer was more nuanced than expected: load factor's raw correlation with average fare was weak (**r = 0.074**), and its regression coefficient was small and negative (**−7.15**) once other features were controlled for. What actually drove fares in this dataset was route distance (the strongest single correlation, **r = 0.553**), whether a route was flagged as saturated (**r = 0.219**, coefficient **+11.77**), and — with by far the largest linear coefficients — the weather, arrival-delay, and NAS delay-share features. That's a genuine, evidence-based finding rather than a foregone conclusion, and it's exactly the kind of result a hypothesis-testing project is supposed to produce: the initial intuition about load factor didn't hold up as the primary driver, but the pipeline surfaced what did.

Building this end to end demonstrates:

- **Handling real, messy, multi-source data at a scale that punishes naive approaches** — chunked reads, mismatched schemas, and inconsistent granularities across five sources had to be reconciled correctly before any modeling could be trusted.
- **Statistical and machine learning literacy beyond calling `.fit()`** — knowing when to use correlation analysis versus regression versus a dimensionality-reduction technique, and evaluating all three against each other with the right metrics (RMSE, MAPE, R², SNR) instead of picking one number and stopping.
- **Engineering for reproducibility and transparency** — a fixed random seed, a fixed feature set, and every intermediate and final artifact (cleaned data, the analysis table, correlation matrices, prediction summaries, feature importances, and a plain-language conclusions file) written to disk so results can be audited rather than taken on faith.
- **Comfort with an ambiguous, real-world result** — the headline hypothesis (load factor drives fares) turned out to be weak, and the project's structure (a dedicated `Hypothesis Signals` section in `Conclusions.txt`) shows that outcome was reported honestly rather than reframed after the fact, which is arguably the most important skill a data project like this can demonstrate.

The main lesson learned was that combining several real government/industry datasets is often harder than the modeling step that follows it — most of the engineering effort in this repository went into `Data.py`'s ingestion and merge logic, not into `Model.py`'s three regression techniques, which is a fairly common (and useful) realization for anyone doing applied data science for the first time.
