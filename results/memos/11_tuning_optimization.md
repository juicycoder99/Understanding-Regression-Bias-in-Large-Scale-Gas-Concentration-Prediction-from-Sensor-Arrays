# 11 - Tuning & Optimization

**Date:** 2026-05-24
**Dataset:** ../data/processed/ethylene_methane.parquet
**Splits artifact:** ../results/tables/validation_splits.parquet
**Subset artifact:** ../data/subsets/fair_subset_indices.parquet

---

## Protocol
- Same 5-fold rolling temporal validation as stages 06-10.
- Training = `train_range ∩ fair_subset` (~395k rows). Val = full val slice.
- Fresh `RobustScaler` per fold.
- Early stopping (boosted/neural): last 15% of training set as internal eval.
- Best config per model: highest mean R2 pooled across folds and targets.
- Total configs evaluated: 2350.

## Models Tuned
- **Pass A (fast):** Ridge, Polynomial, Spline, Huber, ExtraTrees.
- **Pass B (expensive):** LightGBM, CatBoost, HistGradientBoosting, TabNet, MLP.

## Ranking (tuned models, mean across folds & targets)

| model | mae | rmse | r2 |
| --- | --- | --- | --- |
| spline_tuned | 11.8173 | 22.333 | 0.6194 |
| poly_tuned | 11.6617 | 22.4671 | 0.607 |
| ridge_tuned | 12.2421 | 23.1834 | 0.6049 |
| huber_tuned | 11.1076 | 24.4583 | 0.5747 |
| extratrees_tuned | 12.2675 | 23.5594 | 0.5671 |
| mlp_tuned | 14.8562 | 25.3284 | 0.5184 |
| catboost_tuned | 14.1829 | 24.2995 | 0.5143 |
| lgbm_tuned | 13.9956 | 24.4693 | 0.4986 |
| tabnet_tuned | 13.2001 | 24.7656 | 0.4299 |
| hgbr_tuned | 15.9574 | 31.0711 | 0.2909 |

**Best tuned model:** `spline_tuned`

## Best Hyperparameters

| model | config | R2 |
| --- | --- | --- |
| spline_tuned | {'n_knots': 5, 'degree': 3, 'ridge_alpha': 1.0} | 0.6194 |
| poly_tuned | {'degree': 2, 'interaction_only': True, 'ridge_alpha': 10.0} | 0.6070 |
| ridge_tuned | {'alpha': 1.0} | 0.6049 |
| huber_tuned | {'epsilon': 2.0, 'alpha': 0.0001} | 0.5747 |
| extratrees_tuned | {'n_estimators': 200, 'min_samples_leaf': 50, 'max_features': 'sqrt', 'max_depth': 10} | 0.5671 |
| mlp_tuned | {'learning_rate_init': 0.0001, 'hidden_layer_sizes': (128, 64), 'batch_size': 4096, 'alpha': 0.001} | 0.5184 |
| catboost_tuned | {'learning_rate': 0.05, 'l2_leaf_reg': 10, 'depth': 6, 'border_count': 32} | 0.5143 |
| lgbm_tuned | {'subsample': 1.0, 'reg_lambda': 1.0, 'reg_alpha': 1.0, 'num_leaves': 63, 'min_child_samples': 100, 'max_depth': 3, 'learning_rate': 0.03, 'colsample_bytree': 0.7} | 0.4986 |
| tabnet_tuned | {'n_steps': 3, 'n_d_n_a': 8, 'lr': 0.02, 'lambda_sparse': 0.001, 'gamma': 1.0} | 0.4299 |
| hgbr_tuned | {'min_samples_leaf': 100, 'max_leaf_nodes': 31, 'max_depth': None, 'learning_rate': 0.01, 'l2_regularization': 0} | 0.2909 |

## Summary (mean +/- std across 5 folds, per target)

| model | target | mae_mean | mae_std | rmse_mean | rmse_std | r2_mean | r2_std |
| --- | --- | --- | --- | --- | --- | --- | --- |
| catboost_tuned | ethylene_ppm | 2.9137 | 0.5749 | 4.1248 | 0.8194 | 0.4214 | 0.1373 |
| catboost_tuned | methane_ppm | 25.452 | 8.8042 | 44.4742 | 12.8336 | 0.6073 | 0.0527 |
| extratrees_tuned | ethylene_ppm | 2.5691 | 0.5007 | 3.8098 | 0.6295 | 0.5086 | 0.0633 |
| extratrees_tuned | methane_ppm | 21.9658 | 6.7954 | 43.3089 | 12.1362 | 0.6256 | 0.0416 |
| hgbr_tuned | ethylene_ppm | 3.0704 | 0.8795 | 4.5932 | 1.1667 | 0.2978 | 0.1765 |
| hgbr_tuned | methane_ppm | 28.8443 | 11.4687 | 57.549 | 19.6374 | 0.284 | 0.3575 |
| huber_tuned | ethylene_ppm | 1.8418 | 0.3668 | 3.5728 | 0.7008 | 0.5558 | 0.1257 |
| huber_tuned | methane_ppm | 20.3734 | 6.435 | 45.3437 | 13.7247 | 0.5936 | 0.0646 |
| lgbm_tuned | ethylene_ppm | 2.8738 | 0.5159 | 4.1784 | 0.8609 | 0.3927 | 0.2088 |
| lgbm_tuned | methane_ppm | 25.1174 | 8.012 | 44.7603 | 13.4812 | 0.6046 | 0.0535 |
| mlp_tuned | ethylene_ppm | 2.5514 | 0.332 | 3.8377 | 0.5607 | 0.5009 | 0.0483 |
| mlp_tuned | methane_ppm | 27.161 | 7.4584 | 46.8191 | 9.68 | 0.536 | 0.1271 |
| poly_tuned | ethylene_ppm | 2.2924 | 0.5002 | 3.6016 | 0.5826 | 0.5601 | 0.0597 |
| poly_tuned | methane_ppm | 21.031 | 4.9869 | 41.3326 | 10.8677 | 0.6539 | 0.0498 |
| ridge_tuned | ethylene_ppm | 2.111 | 0.3437 | 3.5004 | 0.5659 | 0.5787 | 0.0882 |
| ridge_tuned | methane_ppm | 22.3732 | 5.1726 | 42.8664 | 11.6213 | 0.631 | 0.0449 |
| spline_tuned | ethylene_ppm | 2.1954 | 0.5835 | 3.5072 | 0.7115 | 0.5782 | 0.1043 |
| spline_tuned | methane_ppm | 21.4392 | 5.8228 | 41.1588 | 11.367 | 0.6607 | 0.0416 |
| tabnet_tuned | ethylene_ppm | 2.7984 | 1.0084 | 4.6501 | 1.2841 | 0.2727 | 0.2425 |
| tabnet_tuned | methane_ppm | 23.6019 | 8.3688 | 44.8811 | 9.9573 | 0.5872 | 0.0511 |

## Artifacts
- Per-fold metrics: `../results/tables/11_tuning_metrics_long.parquet`
- Summary metrics: `../results/tables/11_tuning_metrics_summary.parquet`
- Search log: `../results/tables/11_tuning_search_log.parquet`
- Fitted models: `../results/models/tuned_11/fold_{k}_{name}.joblib`
- Figure: `../results/figures/11_tuning_optimization.png`

## Next step
Stage 12: creative prototype model or final reporting.