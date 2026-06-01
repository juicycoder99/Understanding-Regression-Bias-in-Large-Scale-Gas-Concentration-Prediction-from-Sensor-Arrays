# Task 3.3 — Creative Model Direction Proposal

**Date:** 2026-05-28
**Project:** Fresh regression benchmark — methane_ppm and ethylene_ppm from 16-sensor gas array.
**Status:** Approved direction; design document for Notebook 14.
**Closes:** Week 3, Task 3.3 of the One-Month Plan.

---

## Part A — Evidence inventory (Stages 11, 12, 13)

The following findings constrain the choice of direction.

**Performance ceiling and shape.**
- R² ≈ 0.62 ceiling holds under 2,350 tuned configurations + 60 coreset variants. The ceiling is structural.
- Methane is recoverable to R² ≈ 0.66, ethylene only to ≈ 0.59 — a 0.07 asymmetric gap that no method has closed.
- Fold 1 ethylene is a structural collapse point for partition/neural methods (TabNet, LGBM, CatBoost all hit fold-1 ethylene R² ≤ 0.06–0.21).

**Inductive-bias landscape.**
- Smooth global functions (spline, poly, ridge) dominate the leaderboard at all budgets.
- Partition-based methods (ExtraTrees, CatBoost, LGBM) reach methane parity with Ridge (R² ≈ 0.60–0.63) but collapse on ethylene (0.39–0.51).
- Neural methods cannot beat smooth methods at any tested configuration.

**Coreset–model coupling (Stage 13).**
- FPS wins 10/12 cells. Diversity beats representativeness for prediction.
- **Partition methods peak at small budgets:** CatBoost @ 25k, ExtraTrees @ 60k. R² *declines* with more data for these models.
- Smooth methods peak at 395k (monotonic R²(budget) curves).
- Spline is coreset-insensitive (R² range across methods 0.004–0.008).
- CatBoost is coreset-sensitive (R² range across methods 0.027–0.048).

**Cross-protocol audit.** Stage 05 leakage was numerically benign. All Stage 11 reference numbers stand.

**Current benchmark high.** FPS-selected `spline_tuned` at 395k per fold: pooled R² = 0.6221 (methane 0.659, ethylene 0.585).

---

## Part B — Archetype comparison

The PDF lists four candidate archetypes. Each is evaluated against the evidence above.

| archetype | strongest supporting evidence | strongest counter-evidence | expected ceiling | research risk |
|---|---|---|---|---|
| **(1) Coreset-aware spline** | FPS+spline already best (0.6221) | Spline is the *least* coreset-sensitive model (range 0.004–0.008). Formalizing what we already have. | ≤ 0.625 | low gain, low risk |
| **(2) Spline + boosting residual on coreset** | Combines proven winners. Boosting peaks at 25k — cheap residual stage. Complementary inductive biases (smooth + partition). | If residuals are mostly noise (which R² ≈ 0.62 ceiling implies), the residual stage overfits. | 0.625–0.66 if residuals have structure; ≤ 0.62 if they do not | **moderate gain, moderate risk** |
| **(3) Regime-aware model** | 4-way regime structure exists in data; methane/ethylene asymmetry is regime-correlated; fold-1 ethylene collapse may be regime-driven | No direct evidence from Stages 11–13 that regime separation helps. Classification errors propagate. Boundaries are fuzzy. New failure modes. | 0.60–0.65 (highly uncertain) | high gain potential, high risk |
| **(4) Teacher–student compression** | Pergamon deployment focus | The benchmark winner *is already* the lightweight model (spline). No heavy teacher outperforms. Compression presupposes a capacity gap that does not exist here. | ≤ 0.622 (cannot exceed the teacher) | wrong premise |

### Per-archetype verdict

**(1) Coreset-aware spline — REJECTED as primary.** The coreset–spline coupling is the weakest in the benchmark. The headroom (≤ 0.005) does not justify a notebook stage.

**(2) Spline + boosting residual on FPS coreset — RECOMMENDED.** Spline captures the smooth global component (proven champion at 0.622). Residuals = the part of the signal spline cannot model. Boosting is the family best suited to capturing structured local error, and Stage 13 showed it peaks at 25k–60k — a *cheap* residual stage that is unlikely to overfit. Directly targets the methane/ethylene asymmetry: if ethylene-specific residual structure exists, the boosting stage on ethylene residuals can find it.

**(3) Regime-aware — DEFERRED.** Plausible scientifically but unsupported by current evidence. The PDF asks for a proposal "based on the comparative study and the coreset results" — Stages 11–13 contain no regime-vs-non-regime comparison. Could become Week 4+ exploration if the residual approach plateaus.

**(4) Teacher–student compression — REJECTED.** Mechanically the wrong fit. Spline is already small (~112 spline basis weights + Ridge coefficients). No heavy teacher to compress.

---

## Part C — Selected direction

**Spline + boosting residual on FPS coreset.**

### Why it beats the other three

- The only archetype where *every* component is independently validated by Stages 11–13 evidence: spline at 395k as backbone (winner), FPS as coreset (winner), boosting at 25k as residual stage (peak budget per Stage 13).
- Directly addresses the methane/ethylene asymmetry by allowing the residual stage to capture ethylene-specific local structure independently of the backbone.
- Produces a deployment artifact that is still light: spline (cheap) + CatBoost@25k (cheap) = inference cost ≪ 1 ms per sample.
- Its failure mode is interpretable: if pooled R² ≤ 0.62, the conclusion is "the spline residuals are noise" — a publishable negative result.
- Success criteria are quantitative and pre-specified (Section F).

---

# Detailed Design Document

## D. Objective

Build and evaluate a two-stage hybrid model that combines:

1. A frozen `spline_tuned` backbone (Stage 11 config: `SplineTransformer(n_knots=5, degree=3) → Ridge(α=1.0)`) trained on the FPS coreset at 395k per fold, which captures the smooth global component of the target.
2. A boosted residual stage (CatBoost, frozen Stage 11 config) that learns the part of the target the spline cannot represent.

Final prediction: `y_hat = spline_pred(X) + residual_pred(X)`, per target.

## E. Architecture

```
                       ┌─── X (16 scaled sensors)
                       │
              ┌────────┴────────┐
              ▼                 ▼
   ┌──────────────────┐    (same X)
   │ SplineTransformer│
   │ (n_knots=5,deg=3)│
   └────────┬─────────┘
            ▼
   ┌────────────────┐
   │ Ridge(α=1.0)   │  ──► y_spline (shape n×2)
   └────────┬───────┘
            │  during fit:
            │  r_train = y_train − y_spline_train
            ▼
   ┌────────────────────────────────────────┐
   │ Per-target CatBoost residual stage     │
   │   target_1: CatBoost(...).fit(X, r_1)  │
   │   target_2: CatBoost(...).fit(X, r_2)  │
   └────────┬───────────────────────────────┘
            ▼
        y_resid ─── added to y_spline ──► y_hat
```

Frozen configs (no new tuning):
- **Spline:** `SplineTransformer(n_knots=5, degree=3, include_bias=False) → Ridge(alpha=1.0, random_state=42)`.
- **CatBoost residual stage:** `iterations=2000, learning_rate=0.05, depth=6, l2_leaf_reg=10, border_count=32, random_seed=42`; per-target; early stopping with last 15% of training data carved as eval_set, patience=50.

Both stages use the same `RobustScaler` (fit on the spline coreset).

## F. Hypotheses (pre-specified, testable)

- **H1 — Headline:** Pooled R² with hybrid ≥ 0.625 (beats FPS+spline by ≥ 0.003, exits fold-std noise). **Success criterion.**
- **H2 — Per-target:** ethylene R² gain > methane R² gain. The residual stage targets the gap, and ethylene has the larger gap.
- **H3 — Fold-1 ethylene:** R² on fold-1 ethylene improves by ≥ 0.05 over spline alone. If true, the residual stage is capturing the regime structure that spline misses.
- **H4 — Insensitivity transfer:** the hybrid inherits spline's coreset insensitivity — swapping the residual stage's coreset (FPS vs stratified at the same budget) changes pooled R² by ≤ 0.005.

H1 is the pass/fail success criterion. H2–H4 are interpretive — they explain *why* the model works (or fails), feeding directly into Task 4.2.

## G. Training procedure (per fold)

1. Load the 395k FPS coreset for the fold (`coreset_fps_395k_fold{k}.parquet`).
2. Fit `RobustScaler` on the spline coreset's 16 sensor columns; apply to train and val.
3. Fit `spline_tuned` on scaled (X_train, y_train) — single fit, native multi-output.
4. Compute spline residuals on the *training* set: `r_train = y_train − spline.predict(X_train)`.
5. Per target t ∈ {methane, ethylene}:
   - Carve last 15% of coreset rows (temporally latest) as eval_set.
   - Fit CatBoost on (X_main, r_main[:,t]) with eval_set=(X_eval, r_eval[:,t]), early_stopping_rounds=50.
6. Predict on val: `y_hat = spline.predict(X_val) + np.column_stack([cat_t.predict(X_val) for t in targets])`.
7. Record per-target MAE, RMSE, R² against the true y_val.

## H. Required artifacts (consumed)

| artifact | source | usage |
|---|---|---|
| `coreset_fps_395k_fold{k}.parquet` | Stage 12 | primary spline coreset; default residual coreset |
| `coreset_fps_25k_fold{k}.parquet` | Stage 12 | residual-on-small-coreset variant |
| `coreset_fps_60k_fold{k}.parquet` | Stage 12 | ExtraTrees-residual variant |
| `coreset_stratified_395k_fold{k}.parquet` | Stage 12 | ablation: residual on non-FPS coreset |
| `validation_splits.parquet` | Stage 03 | val slices, untouched |
| `data/processed/ethylene_methane.parquet` | upstream | full data |

## I. Variants (5 main + 1 sanity)

| variant | backbone coreset | residual model | residual coreset | rationale |
|---|---|---|---|---|
| V0 | FPS 395k | (none — baseline) | — | reproduce Stage 13's spline winner |
| **V1 (primary)** | FPS 395k | CatBoost | FPS 395k | hybrid on a single training pool |
| V2 | FPS 395k | CatBoost | FPS 25k | residual at CatBoost's peak budget |
| V3 | FPS 395k | ExtraTrees | FPS 60k | residual stage swap (ExtraTrees @ peak) |
| V4 | FPS 395k | CatBoost | stratified 395k | coreset-attribution ablation (tests H4) |
| Sanity | — | CatBoost stand-alone (Stage 11 config) | FPS 25k | reproduces Stage 13's CatBoost winner |

## J. Analyses (preempt Task 4.2)

1. **A — Ingredient attribution.** Compare V0 → V1 (backbone + residual gain) and V1 → V4 (coreset effect on residual).
2. **B — Per-target structure.** Does the residual gain accrue to ethylene more than methane? (H2)
3. **C — Per-fold breakdown.** Is fold-1 ethylene R² improved by ≥ 0.05? (H3)
4. **D — Residual-model swap.** V1 vs V3. CatBoost vs ExtraTrees residual.

## K. Evaluation protocol

- 5 rolling temporal folds, same as Stages 06–13.
- Val slice = full untouched val window per fold.
- Metrics per target per fold: MAE, RMSE, R².
- Aggregation: mean ± std across folds.
- No new tuning; all configs frozen.

## L. Expected outputs

**Tables (`results/tables/`):**
- `14_prototype_metrics_long.parquet` — 6 variants × 5 folds × 2 targets = 60 rows.
- `14_prototype_metrics_summary.parquet` — fold-aggregated.
- `14_prototype_ablation.parquet` — per-analysis (A–D) deltas.
- `14_per_fold_breakdown.parquet` — per-(variant, fold, target) for Analysis C.

**Figures (`results/figures/`):**
- `14_prototype_vs_baselines.png` — pooled R² bar chart, 6 variants.
- `14_per_fold_comparison.png` — V0 vs V1 per fold per target.
- `14_residual_structure.png` — residual scatter (true_resid vs predicted_resid, fold 1).
- `14_per_target_breakdown.png` — per-variant per-target bars.

**Models (`results/models/prototype_14/`):**
- `fold_{k}_V1_spline.joblib`, `fold_{k}_V1_resid_methane.joblib`, `fold_{k}_V1_resid_ethylene.joblib`.
- Only the primary variant V1 has models saved.

**Memo (`results/memos/14_prototype_v1.md`):**
- Hypotheses H1–H4 with pass/fail verdicts.
- Analyses A–D with quantitative results.
- Recommendation for Task 4.1 graduation.

## M. Runtime estimate

| component | per fold | total over 5 folds |
|---|---|---|
| Spline fit + predict (395k) | ~5 s | 25 s |
| CatBoost residual fit per target (395k, V1) | ~10–20 s | × 2 × 5 ≈ 150 s |
| CatBoost residual fit per target (25k, V2) | ~3–5 s | × 2 × 5 ≈ 50 s |
| ExtraTrees residual fit per target (60k, V3) | ~10 s | × 2 × 5 ≈ 100 s |
| CatBoost residual fit per target (stratified 395k, V4) | ~10–20 s | × 2 × 5 ≈ 150 s |
| CatBoost stand-alone (Sanity) | ~3–5 s | × 2 × 5 ≈ 50 s |
| Aggregations + figures + memo | — | ~5–10 min |
| **Wall-clock total** | — | **~20–30 min CPU** |

## N. What this task does NOT do

- Does not introduce a regime classifier (Option 3, deferred).
- Does not introduce hyperparameter search — all components frozen at Stage 11 best.
- Does not introduce new metrics beyond MAE / RMSE / R².
- Does not save artifacts for non-primary variants.
- Does not graduate to Task 4.1 automatically — the memo's recommendation triggers Task 4.1 if H1 passes.

## O. Notebook structure (Notebook 14)

| section | content |
|---|---|
| 1. Header | Direction selected, hypotheses, frozen configs, no-tuning rule |
| 2. Imports | Paths, SEED=42, variant list, frozen Stage 11 configs |
| 3. Load + cache | Data, splits, val cache, Stage 11 + Stage 13 reference metrics |
| 4. Utilities | `load_coreset()`, `temporal_carve()`, `record_metrics()` |
| 5. Model factories | `make_spline()`, `make_catboost_residual()`, `make_extratrees_residual()` |
| 6. Hybrid predict | `fit_hybrid(coreset_backbone, coreset_residual, residual_model)` → returns (spline, residual_models) |
| 7. Variant runner | Loop over V0, V1, V2, V3, V4, Sanity × 5 folds |
| 8. Save tables | Long + summary parquets |
| 9. Analysis A | Ingredient attribution (V0→V1, V1→V4 deltas) |
| 10. Analysis B | Per-target gain comparison (H2) |
| 11. Analysis C | Per-fold breakdown (H3) |
| 12. Analysis D | Residual-model swap (V1 vs V3) |
| 13. Verdicts | Pass/fail on H1–H4 |
| 14. Figures | 4 figures listed above |
| 15. Memo | Full assessment with Task 4.1 recommendation |

---

**Sign-off.** Direction approved. Notebook 14 (`14_prototype_v1.ipynb`) implements the design above.
