# 09 - Boosted Models (untuned)

**Date:** 2026-05-23
**Dataset:** ../data/processed/ethylene_methane.parquet
**Splits artifact:** ../results/tables/validation_splits.parquet
**Subset artifact:** ../data/subsets/fair_subset_indices.parquet

---

## Protocol
- Per fold: training set = `train_range ∩ fair_subset` (~395k rows). Validation = full val slice (untouched).
- Fresh `RobustScaler` fit on each fold's training intersection; applied to train and val.
- All models wrapped in `MultiOutputRegressor` (one fit per target). This is the documented deviation from 06–08.
- 5 folds × 5 models × 2 targets = 50 metric records.

## Roster
- `gbr`      – `GradientBoostingRegressor(n_estimators=100, max_depth=3, lr=0.1)`, sklearn classical.
- `hgbr`     – `HistGradientBoostingRegressor(max_iter=100, lr=0.1)`, sklearn histogram.
- `xgb`      – `XGBRegressor(n_estimators=100, max_depth=6, lr=0.3)`, xgboost.
- `lgbm`     – `LGBMRegressor(n_estimators=100, lr=0.1)`, lightgbm.
- `catboost` – `CatBoostRegressor(iterations=100, depth=6, lr=0.1)`, catboost.

## Ranking (mean across folds & targets, sorted by R² descending)

Note: MAE/RMSE pool across two targets with different scales; R² is the scale-free comparator.

| model | mae | rmse | r2 |
| --- | --- | --- | --- |
| gbr | 14.5889 | 28.0801 | 0.3736 |
| catboost | 15.2446 | 29.078 | 0.3532 |
| lgbm | 15.0 | 29.6296 | 0.3268 |
| hgbr | 15.5422 | 30.6774 | 0.3252 |
| xgb | 18.6909 | 35.0624 | 0.0657 |

**Best boosted model:** `gbr`

## Summary (mean ± std across 5 folds, per target)

| model | target | mae_mean | mae_std | rmse_mean | rmse_std | r2_mean | r2_std |
| --- | --- | --- | --- | --- | --- | --- | --- |
| catboost | ethylene_ppm | 3.175 | 0.8131 | 4.5076 | 1.0676 | 0.3237 | 0.1451 |
| catboost | methane_ppm | 27.3141 | 9.1248 | 53.6484 | 16.5989 | 0.3827 | 0.2897 |
| gbr | ethylene_ppm | 3.1509 | 0.6814 | 4.6013 | 0.7813 | 0.2738 | 0.1554 |
| gbr | methane_ppm | 26.027 | 10.3826 | 51.5589 | 18.7744 | 0.4734 | 0.186 |
| hgbr | ethylene_ppm | 2.9766 | 0.7466 | 4.4763 | 1.0029 | 0.332 | 0.1285 |
| hgbr | methane_ppm | 28.1077 | 12.1062 | 56.8785 | 21.3193 | 0.3185 | 0.3561 |
| lgbm | ethylene_ppm | 3.074 | 0.8686 | 4.6276 | 1.1805 | 0.2855 | 0.1904 |
| lgbm | methane_ppm | 26.926 | 11.0349 | 54.6316 | 18.9724 | 0.3681 | 0.3087 |
| xgb | ethylene_ppm | 3.3721 | 1.0103 | 5.0038 | 1.1695 | 0.1706 | 0.1575 |
| xgb | methane_ppm | 34.0097 | 12.7928 | 65.121 | 21.1702 | -0.0391 | 0.7371 |

## Artifacts
- Per-fold metrics: `../results/tables/09_boosted_metrics_long.parquet`
- Summary metrics:  `../results/tables/09_boosted_metrics_summary.parquet`
- Fitted models:    `../results/models/boosted_09/fold_{k}_{name}.joblib` (25 files)
- Figure:           `../results/figures/09_boosted_models.png`

## Next step
Notebook `10_neural_models.ipynb` (pending approval): MLP and TabNet under the same protocol.