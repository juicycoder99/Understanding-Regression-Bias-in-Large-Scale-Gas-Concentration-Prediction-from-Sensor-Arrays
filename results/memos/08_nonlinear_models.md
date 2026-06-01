# 08 - Nonlinear Models (untuned)

**Date:** 2026-05-23
**Dataset:** ../data/processed/ethylene_methane.parquet
**Splits artifact:** ../results/tables/validation_splits.parquet
**Subset artifact:** ../data/subsets/fair_subset_indices.parquet

---

## Protocol
- Per fold: training set = `train_range ∩ fair_subset` (~395k rows). Validation = full val slice (untouched).
- Fresh `RobustScaler` fit on each fold's training intersection; applied to train and val.
- All models native multi-output (poly/spline via LinearRegression on expanded basis).
- 5 folds × 6 models × 2 targets = 60 metric records.

## Roster
- `knn`        – `KNeighborsRegressor(n_neighbors=15)`, local nonlinear.
- `poly`       – `PolynomialFeatures(degree=2) → LinearRegression`, ~153 features.
- `spline`     – `SplineTransformer(n_knots=5, degree=3) → LinearRegression`, ~112 features.
- `tree`       – `DecisionTreeRegressor(max_depth=12)`, capped depth.
- `rf`         – `RandomForestRegressor(n_estimators=200, min_samples_leaf=20)`.
- `extratrees` – `ExtraTreesRegressor(n_estimators=200, min_samples_leaf=20)`.

## Ranking (mean across folds & targets, sorted by R² descending)

Note: MAE/RMSE pool across two targets with different scales; R² is the scale-free comparator.

| model | mae | rmse | r2 |
| --- | --- | --- | --- |
| poly | 11.7009 | 22.278 | 0.5997 |
| spline | 12.2015 | 22.9384 | 0.5911 |
| extratrees | 14.703 | 27.5891 | 0.4345 |
| rf | 16.9902 | 33.5135 | 0.2018 |
| knn | 17.3572 | 33.7179 | 0.1193 |
| tree | 18.5502 | 39.7925 | -0.0181 |

**Best nonlinear model:** `poly`

## Summary (mean ± std across 5 folds, per target)

| model | target | mae_mean | mae_std | rmse_mean | rmse_std | r2_mean | r2_std |
| --- | --- | --- | --- | --- | --- | --- | --- |
| extratrees | ethylene_ppm | 2.5635 | 0.879 | 4.2238 | 1.0255 | 0.4044 | 0.1369 |
| extratrees | methane_ppm | 26.8424 | 10.5285 | 50.9543 | 14.6346 | 0.4645 | 0.1577 |
| knn | ethylene_ppm | 3.1003 | 0.9762 | 5.2832 | 1.1716 | 0.074 | 0.1353 |
| knn | methane_ppm | 31.614 | 18.8946 | 62.1526 | 25.7664 | 0.1646 | 0.5924 |
| poly | ethylene_ppm | 2.4363 | 0.5816 | 3.7143 | 0.6279 | 0.5351 | 0.0448 |
| poly | methane_ppm | 20.9654 | 5.5581 | 40.8417 | 11.0221 | 0.6643 | 0.0435 |
| rf | ethylene_ppm | 2.6328 | 0.9812 | 4.641 | 1.1891 | 0.2804 | 0.1954 |
| rf | methane_ppm | 31.3475 | 11.6394 | 62.386 | 16.4445 | 0.1232 | 0.4525 |
| spline | ethylene_ppm | 2.2877 | 0.5373 | 3.6652 | 0.6586 | 0.5409 | 0.1002 |
| spline | methane_ppm | 22.1153 | 5.588 | 42.2117 | 10.7994 | 0.6414 | 0.0264 |
| tree | ethylene_ppm | 2.7117 | 0.6958 | 4.8315 | 0.9085 | 0.2154 | 0.0964 |
| tree | methane_ppm | 34.3887 | 13.8641 | 74.7536 | 17.8759 | -0.2516 | 0.6189 |

## Artifacts
- Per-fold metrics: `../results/tables/08_nonlinear_metrics_long.parquet`
- Summary metrics:  `../results/tables/08_nonlinear_metrics_summary.parquet`
- Fitted models:    `../results/models/nonlinear_08/fold_{k}_{name}.joblib` (30 files)
- Figure:           `../results/figures/08_nonlinear_models.png`

## Next step
Notebook `09_boosted_models.ipynb` (pending approval): GradientBoosting, HistGradientBoosting, XGBoost/LightGBM/CatBoost under the same protocol.