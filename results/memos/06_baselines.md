# 06 - Baselines (Mean / Linear / Ridge)

**Date:** 2026-05-22
**Dataset:** ../data/processed/ethylene_methane.parquet
**Splits artifact:** ../results/tables/validation_splits.parquet
**Subset artifact:** ../data/subsets/fair_subset_indices.parquet

---

## Protocol
- Per fold: training set = `train_range ∩ fair_subset` (~395k rows). Validation = full val slice (untouched).
- Fresh `RobustScaler` fit on each fold's training intersection; applied to train and val.
- Baselines fit natively on the (n, 2) target matrix; metrics computed per target.
- 5 folds × 3 models × 2 targets = 30 metric records.

## Roster
- `mean`   – `DummyRegressor(strategy=mean)` per target (sanity floor).
- `linear` – `LinearRegression()`, native multi-output.
- `ridge`  – `Ridge(alpha=1.0)`, native multi-output.

## Ranking (mean across folds & targets, sorted by R² descending)

Note: MAE/RMSE pool across two targets with different scales; R² is the scale-free comparator.

| model | mae | rmse | r2 |
| --- | --- | --- | --- |
| ridge | 12.2421 | 23.1834 | 0.6049 |
| linear | 12.2734 | 23.1946 | 0.6047 |
| mean | 35.6948 | 40.7602 | -0.2269 |

**Best baseline:** `ridge`

## Summary (mean ± std across 5 folds, per target)

| model | target | mae_mean | mae_std | rmse_mean | rmse_std | r2_mean | r2_std |
| --- | --- | --- | --- | --- | --- | --- | --- |
| linear | ethylene_ppm | 2.1129 | 0.3405 | 3.4989 | 0.5656 | 0.5789 | 0.089 |
| linear | methane_ppm | 22.4339 | 5.1652 | 42.8904 | 11.6172 | 0.6305 | 0.0451 |
| mean | ethylene_ppm | 5.2144 | 0.5109 | 5.8209 | 0.6881 | -0.1646 | 0.1957 |
| mean | methane_ppm | 66.1753 | 6.2891 | 75.6996 | 9.4999 | -0.2891 | 0.6015 |
| ridge | ethylene_ppm | 2.111 | 0.3437 | 3.5004 | 0.5659 | 0.5787 | 0.0882 |
| ridge | methane_ppm | 22.3732 | 5.1726 | 42.8664 | 11.6213 | 0.631 | 0.0449 |

## Artifacts
- Per-fold metrics: `../results/tables/06_baseline_metrics_long.parquet`
- Summary metrics:  `../results/tables/06_baseline_metrics_summary.parquet`
- Fitted models:    `../results/models/baselines/fold_{k}_{name}.joblib` (15 files)
- Figure:           `../results/figures/06_baselines.png`

## Next step
Notebook `07_models.ipynb` (pending approval): non-linear models (tree / boosted) under the same protocol.