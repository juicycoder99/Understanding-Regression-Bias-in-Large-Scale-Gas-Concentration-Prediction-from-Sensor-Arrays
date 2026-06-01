# 10 - Neural Models (untuned)

**Date:** 2026-05-23
**Dataset:** ../data/processed/ethylene_methane.parquet
**Splits artifact:** ../results/tables/validation_splits.parquet
**Subset artifact:** ../data/subsets/fair_subset_indices.parquet

---

## Protocol
- Per fold: training set = `train_range ∩ fair_subset` (~395k rows). Validation = full val slice (untouched).
- Fresh `RobustScaler` fit on each fold's training intersection; applied to train and val.
- MLP models: native multi-output, early stopping with 10% internal validation.
- TabNet: per-target fit (two models per fold), early stopping with 15% internal validation (patience=15).
- 5 folds × 3 models × 2 targets = 30 metric records.

## Roster
- `mlp_small`  – `MLPRegressor(hidden_layer_sizes=(64, 32))`, Adam, ReLU, early stopping.
- `mlp_medium` – `MLPRegressor(hidden_layer_sizes=(256, 128, 64))`, Adam, ReLU, early stopping.
- `tabnet`     – `TabNetRegressor(n_d=8, n_a=8, n_steps=3)`, per-target, patience-based.

## Ranking (mean across folds & targets, sorted by R² descending)

| model | mae | rmse | r2 |
| --- | --- | --- | --- |
| tabnet | 17.6639 | 33.4821 | 0.2135 |
| mlp_medium | 19.5781 | 34.3654 | 0.069 |
| mlp_small | 23.8947 | 38.802 | 0.0288 |

**Best neural model:** `tabnet`

## Summary (mean ± std across 5 folds, per target)

| model | target | mae_mean | mae_std | rmse_mean | rmse_std | r2_mean | r2_std |
| --- | --- | --- | --- | --- | --- | --- | --- |
| mlp_medium | ethylene_ppm | 3.9101 | 1.1093 | 5.284 | 1.3154 | 0.0644 | 0.2385 |
| mlp_medium | methane_ppm | 35.2461 | 12.4527 | 63.4467 | 17.8545 | 0.0736 | 0.5312 |
| mlp_small | ethylene_ppm | 2.7099 | 0.7928 | 4.1513 | 0.9838 | 0.411 | 0.1995 |
| mlp_small | methane_ppm | 45.0796 | 15.8955 | 73.4528 | 25.6316 | -0.3535 | 1.0243 |
| tabnet | ethylene_ppm | 2.8129 | 0.9278 | 4.5819 | 1.1375 | 0.287 | 0.2574 |
| tabnet | methane_ppm | 32.515 | 8.0149 | 62.3823 | 17.974 | 0.1401 | 0.3869 |

## Artifacts
- Per-fold metrics: `../results/tables/10_neural_metrics_long.parquet`
- Summary metrics:  `../results/tables/10_neural_metrics_summary.parquet`
- Fitted models:    `../results/models/neural_10/fold_{k}_{name}.joblib`
- Figure:           `../results/figures/10_neural_models.png`

## Next step
Stage 11 (`11_tuning_optimization.ipynb`): hyperparameter tuning of selected candidates from stages 06–10.