# 07 - Linear / Statistical Models (untuned)

**Date:** 2026-05-23
**Dataset:** ../data/processed/ethylene_methane.parquet
**Splits artifact:** ../results/tables/validation_splits.parquet
**Subset artifact:** ../data/subsets/fair_subset_indices.parquet

---

## Protocol
- Per fold: training set = `train_range ∩ fair_subset` (~395k rows). Validation = full val slice (untouched).
- Fresh `RobustScaler` fit on each fold's training intersection; applied to train and val.
- All models at sklearn defaults; no tuning (tuning is stage 11).
- 5 folds × 6 models × 2 targets = 60 metric records.

## Roster
- `lasso`                  – `Lasso(alpha=1.0)`, native per-target solve.
- `elastic_net`            – `ElasticNet(alpha=1.0, l1_ratio=0.5)`, native per-target solve.
- `multitask_lasso`        – `MultiTaskLasso(alpha=1.0)`, joint sparsity across targets.
- `multitask_elastic_net`  – `MultiTaskElasticNet(alpha=1.0, l1_ratio=0.5)`, joint L1+L2 across targets.
- `pls`                    – `PLSRegression(n_components=2)`, joint projection.
- `huber`                  – `MultiOutputRegressor(HuberRegressor())`, robust linear (per-target wrap).

## Ranking (mean across folds & targets, sorted by R² descending)

Note: MAE/RMSE pool across two targets with different scales; R² is the scale-free comparator.

| model | mae | rmse | r2 |
| --- | --- | --- | --- |
| huber | 11.0138 | 24.5064 | 0.5643 |
| multitask_lasso | 14.4044 | 25.6789 | 0.4583 |
| pls | 15.8237 | 26.9209 | 0.4412 |
| lasso | 14.6645 | 25.8513 | 0.4022 |
| multitask_elastic_net | 18.5238 | 27.7326 | 0.402 |
| elastic_net | 18.7177 | 27.8762 | 0.3586 |

**Best linear model:** `huber`

## Summary (mean ± std across 5 folds, per target)

| model | target | mae_mean | mae_std | rmse_mean | rmse_std | r2_mean | r2_std |
| --- | --- | --- | --- | --- | --- | --- | --- |
| elastic_net | ethylene_ppm | 3.8878 | 0.4967 | 4.708 | 0.6091 | 0.2405 | 0.1185 |
| elastic_net | methane_ppm | 33.5475 | 8.2316 | 51.0445 | 13.6753 | 0.4767 | 0.0604 |
| huber | ethylene_ppm | 1.8631 | 0.3408 | 3.6722 | 0.6414 | 0.5324 | 0.1194 |
| huber | methane_ppm | 20.1646 | 7.6771 | 45.3406 | 14.2884 | 0.5962 | 0.0764 |
| lasso | ethylene_ppm | 3.9356 | 0.4898 | 4.7216 | 0.5917 | 0.2382 | 0.0984 |
| lasso | methane_ppm | 25.3934 | 9.6695 | 46.981 | 14.7406 | 0.5662 | 0.0828 |
| multitask_elastic_net | ethylene_ppm | 3.5061 | 0.5445 | 4.4245 | 0.6937 | 0.3272 | 0.1361 |
| multitask_elastic_net | methane_ppm | 33.5414 | 8.2338 | 51.0406 | 13.6792 | 0.4768 | 0.0603 |
| multitask_lasso | ethylene_ppm | 3.3562 | 0.49 | 4.3346 | 0.7254 | 0.3512 | 0.1513 |
| multitask_lasso | methane_ppm | 25.4525 | 9.6459 | 47.0231 | 14.7383 | 0.5653 | 0.0831 |
| pls | ethylene_ppm | 3.001 | 0.6359 | 4.2849 | 0.8361 | 0.3654 | 0.165 |
| pls | methane_ppm | 28.6465 | 10.4168 | 49.557 | 15.4011 | 0.5171 | 0.086 |

## Artifacts
- Per-fold metrics: `../results/tables/07_linear_metrics_long.parquet`
- Summary metrics:  `../results/tables/07_linear_metrics_summary.parquet`
- Fitted models:    `../results/models/linear_07/fold_{k}_{name}.joblib` (30 files)
- Figure:           `../results/figures/07_linear_models.png`

## Next step
Notebook `08_nonlinear_models.ipynb` (pending approval): KNN, Polynomial, Spline, DecisionTree, RandomForest, ExtraTrees under the same protocol.