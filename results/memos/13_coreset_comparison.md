# 13 - Coreset Comparison on the Regression Task

**Date:** 2026-05-28
**Dataset:** ../data/processed/ethylene_methane.parquet
**Splits artifact:** ../results/tables/validation_splits.parquet
**Coreset directory:** ../data/subsets

---

## Protocol
- 5 rolling temporal folds, same as Stages 06-11. Val slice unchanged.
- Fresh `RobustScaler` per (method, budget, fold), fit on the coreset only.
- Three frozen Stage 11 models: spline_tuned, extratrees_tuned, catboost_tuned. No new tuning.
- CatBoost early stopping: last 15% of each coreset (temporal carve), patience=50.
- Stage 11 reference numbers loaded from `11_tuning_metrics_long.parquet` (not re-computed).
- Total records produced: 600.

**Best (method, budget, model) overall:** `fps` at budget 395k for `spline_tuned` — pooled R² = 0.6221.

## Analysis A — Sensitivity

| model | budget_label | best_method | best_r2 | worst_method | worst_r2 | r2_range | best_vs_runner |
| --- | --- | --- | --- | --- | --- | --- | --- |
| spline_tuned | 25k | fps | 0.6166 | random | 0.6082 | 0.0084 | 0.0019 |
| spline_tuned | 60k | fps | 0.62 | kmeans | 0.6148 | 0.0052 | 0.0025 |
| spline_tuned | 120k | fps | 0.6218 | kmeans | 0.6167 | 0.0051 | 0.0015 |
| spline_tuned | 395k | fps | 0.6221 | kmeans | 0.618 | 0.0041 | 0.0025 |
| extratrees_tuned | 25k | fps | 0.578 | stratified | 0.5664 | 0.0116 | 0.0015 |
| extratrees_tuned | 60k | fps | 0.5834 | density | 0.5738 | 0.0096 | 0.0068 |
| extratrees_tuned | 120k | fps | 0.5814 | stratified | 0.5729 | 0.0085 | 0.007 |
| extratrees_tuned | 395k | kmeans | 0.5727 | density | 0.5658 | 0.0069 | 0.0031 |
| catboost_tuned | 25k | fps | 0.5275 | density | 0.48 | 0.0475 | 0.0104 |
| catboost_tuned | 60k | fps | 0.5248 | density | 0.4834 | 0.0414 | 0.0103 |
| catboost_tuned | 120k | random | 0.5134 | density | 0.4786 | 0.0348 | 0.0035 |
| catboost_tuned | 395k | fps | 0.5148 | kmeans | 0.4881 | 0.0267 | 0.0051 |

**Per-model mean sensitivity (avg of R² range across 4 budgets):**

| model | mean_r2_range_across_budgets |
| --- | --- |
| catboost_tuned | 0.0376 |
| extratrees_tuned | 0.0092 |
| spline_tuned | 0.0057 |

## Analysis B — Winning Coreset Method

| model | budget_label | best_method | best_r2 | runner_up_method | runner_up_r2 | best_vs_runner |
| --- | --- | --- | --- | --- | --- | --- |
| spline_tuned | 25k | fps | 0.6166 | density | 0.6147 | 0.0019 |
| spline_tuned | 60k | fps | 0.62 | stratified | 0.6175 | 0.0025 |
| spline_tuned | 120k | fps | 0.6218 | density | 0.6203 | 0.0015 |
| spline_tuned | 395k | fps | 0.6221 | stratified | 0.6196 | 0.0025 |
| extratrees_tuned | 25k | fps | 0.578 | density | 0.5765 | 0.0015 |
| extratrees_tuned | 60k | fps | 0.5834 | kmeans | 0.5766 | 0.0068 |
| extratrees_tuned | 120k | fps | 0.5814 | random | 0.5744 | 0.007 |
| extratrees_tuned | 395k | kmeans | 0.5727 | fps | 0.5696 | 0.0031 |
| catboost_tuned | 25k | fps | 0.5275 | stratified | 0.5171 | 0.0104 |
| catboost_tuned | 60k | fps | 0.5248 | random | 0.5145 | 0.0103 |
| catboost_tuned | 120k | random | 0.5134 | stratified | 0.5099 | 0.0035 |
| catboost_tuned | 395k | fps | 0.5148 | random | 0.5097 | 0.0051 |

**Winner counts across 12 cells:**

| method | wins |
| --- | --- |
| fps | 10 |
| kmeans | 1 |
| random | 1 |

## Analysis C — Budget-Quality Curves

| model | method | r2_25k | r2_60k | r2_120k | r2_395k | total_gain | last_step_gain | monotonic |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| spline_tuned | random | 0.6082 | 0.6165 | 0.6195 | 0.6192 | 0.011 | -0.0003 | True |
| spline_tuned | kmeans | 0.6095 | 0.6148 | 0.6167 | 0.618 | 0.0085 | 0.0013 | True |
| spline_tuned | fps | 0.6166 | 0.62 | 0.6218 | 0.6221 | 0.0055 | 0.0003 | True |
| spline_tuned | density | 0.6147 | 0.6164 | 0.6203 | 0.6186 | 0.0039 | -0.0017 | True |
| spline_tuned | stratified | 0.6127 | 0.6175 | 0.6197 | 0.6196 | 0.0069 | -0.0001 | True |
| extratrees_tuned | random | 0.5667 | 0.575 | 0.5744 | 0.5665 | -0.0002 | -0.0079 | False |
| extratrees_tuned | kmeans | 0.5687 | 0.5766 | 0.5734 | 0.5727 | 0.004 | -0.0007 | True |
| extratrees_tuned | fps | 0.578 | 0.5834 | 0.5814 | 0.5696 | -0.0084 | -0.0118 | False |
| extratrees_tuned | density | 0.5765 | 0.5738 | 0.5733 | 0.5658 | -0.0107 | -0.0075 | False |
| extratrees_tuned | stratified | 0.5664 | 0.5741 | 0.5729 | 0.5668 | 0.0004 | -0.0061 | False |
| catboost_tuned | random | 0.511 | 0.5145 | 0.5134 | 0.5097 | -0.0013 | -0.0037 | True |
| catboost_tuned | kmeans | 0.5121 | 0.4976 | 0.4968 | 0.4881 | -0.024 | -0.0087 | False |
| catboost_tuned | fps | 0.5275 | 0.5248 | 0.5059 | 0.5148 | -0.0127 | 0.0089 | False |
| catboost_tuned | density | 0.48 | 0.4834 | 0.4786 | 0.5077 | 0.0277 | 0.0291 | True |
| catboost_tuned | stratified | 0.5171 | 0.4988 | 0.5099 | 0.5028 | -0.0143 | -0.0071 | False |

## Analysis D — Cross-Protocol Audit

Compares per-fold-stratified at budget=395k (this protocol, leakage-clean) against the Stage 11 reference (Stage 05 1M global stratified, intersected per fold — mild construction-pool leakage).

| model | r2_new_per_fold_strat_395k | r2_new_std | r2_stage11_reference | r2_stage11_ref_std | abs_gap | combined_std | gap_within_std |
| --- | --- | --- | --- | --- | --- | --- | --- |
| spline_tuned | 0.6196 | 0.0864 | 0.6194 | 0.0866 | 0.0001 | 0.0865 | True |
| extratrees_tuned | 0.5668 | 0.0763 | 0.5671 | 0.0797 | 0.0003 | 0.078 | True |
| catboost_tuned | 0.5028 | 0.1491 | 0.5143 | 0.1387 | 0.0115 | 0.144 | True |

**Verdict:** All gaps within combined fold std — Stage 05 leakage was numerically benign at the 1M budget for these models.

## Artifacts
- Long records: `../results/tables/13_coreset_comparison_long.parquet`
- Summary: `../results/tables/13_coreset_comparison_summary.parquet`
- Sensitivity: `../results/tables/13_coreset_sensitivity.parquet`
- Budget curves: `../results/tables/13_budget_curves.parquet`
- Cross-protocol audit: `../results/tables/13_cross_protocol_audit.parquet`
- Figures: `../results/figures/13_{method_ranking,budget_curves,sensitivity_heatmap,per_target_breakdown}.png`

## Inputs to Task 3.3 (Creative Direction Proposal)

- The winning (method, budget, model) combinations indicate which coreset structure best supports each modeling regime.
- The sensitivity ranking identifies whether a coreset-aware prototype should be built on a coreset-sensitive backbone (high payoff from good selection) or on a robust backbone (defensive choice).
- The budget curves show whether smaller training budgets are viable for deployment, informing the compute-cost framing of the creative direction.
- The cross-protocol audit confirms (or revises) whether Stage 11 baselines can be cited directly or need re-statement.

## Next step

Task 3.3: write the creative model direction proposal based on the analyses above.