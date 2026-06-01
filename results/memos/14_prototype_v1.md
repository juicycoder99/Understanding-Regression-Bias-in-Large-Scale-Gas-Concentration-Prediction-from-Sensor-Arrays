# 14 - Prototype v1: Spline + Boosting Residual on FPS Coreset

**Date:** 2026-05-29
**Dataset:** ../data/processed/ethylene_methane.parquet
**Splits artifact:** ../results/tables/validation_splits.parquet

---

## Direction
Two-stage hybrid: spline_tuned backbone trained on FPS 395k coreset + per-target CatBoost residual stage. Frozen Stage 11 configs.

## Hypotheses (pre-specified)
- **H1:** pooled R² >= 0.625.
- **H2:** ethylene R² gain > methane R² gain over spline-only.
- **H3:** fold-1 ethylene R² improves by >= 0.05 vs spline-only.
- **H4:** |V1 - V4| <= 0.005.

## Variants

| variant | backbone | residual | resid_coreset |
| --- | --- | --- | --- |
| V0_spline_only | fps_395k | (none) | (none) |
| V1_spline_cat_fps395k | fps_395k | catboost | fps_395k |
| V2_spline_cat_fps25k | fps_395k | catboost | fps_25k |
| V3_spline_et_fps60k | fps_395k | extratrees | fps_60k |
| V4_spline_cat_strat395k | fps_395k | catboost | stratified_395k |
| Sanity_cat_only_fps25k | (none) | catboost | fps_25k |

## Pooled R² by Variant

| variant | pooled_r2_mean | pooled_r2_std |
| --- | --- | --- |
| V3_spline_et_fps60k | 0.6236 | 0.0813 |
| V0_spline_only | 0.6221 | 0.0829 |
| V1_spline_cat_fps395k | 0.606 | 0.1104 |
| V2_spline_cat_fps25k | 0.6056 | 0.1169 |
| V4_spline_cat_strat395k | 0.5951 | 0.1272 |
| Sanity_cat_only_fps25k | 0.5275 | 0.123 |

## Analysis A — Ingredient Attribution

| analysis | comparison | delta_r2 |
| --- | --- | --- |
| A — backbone + residual gain | V1 − V0 | -0.016 |
| A — coreset effect on residual | V1 − V4 (FPS − stratified residual) | 0.0109 |
| A — residual budget effect | V1 − V2 (395k − 25k residual) | 0.0004 |
| D — residual-model swap | V1 − V3 (CatBoost − ExtraTrees residual) | -0.0176 |

## Verdicts on H1-H4

| hypothesis | value | threshold | verdict |
| --- | --- | --- | --- |
| H1 (pooled R² ≥ 0.625) | 0.606 | 0.625 | FAIL |
| H2 (ethylene gain > methane gain) | meth -0.0054, eth -0.0267 | eth > meth | FAIL |
| H3 (fold-1 ethylene gain ≥ 0.05) | -0.1343 | 0.05 | FAIL |
| H4 (|V1 − V4| ≤ 0.005) | 0.0109 | 0.005 | FAIL |

**Overall recommendation:** DO NOT graduate (consider regime-aware alternative)

## Artifacts
- Long records: `../results/tables/14_prototype_metrics_long.parquet`
- Summary: `../results/tables/14_prototype_metrics_summary.parquet`
- Ablation: `../results/tables/14_prototype_ablation.parquet`
- Per-fold breakdown: `../results/tables/14_per_fold_breakdown.parquet`
- Models (V1 only): `../results/models/prototype_14/fold_{k}_V1_spline_cat_fps395k.joblib`
- Figures: `../results/figures/14_{prototype_vs_baselines,per_fold_comparison,residual_structure,per_target_breakdown}.png`

## Next step

If H1 passed: Task 4.1 graduates V1 to the final prototype slot. Proceed to Task 4.2 interpretation (already preempted by Analyses A-C here).

If H1 failed: revisit Task 3.3 archetype choice. The regime-aware alternative (deferred) becomes the next candidate.