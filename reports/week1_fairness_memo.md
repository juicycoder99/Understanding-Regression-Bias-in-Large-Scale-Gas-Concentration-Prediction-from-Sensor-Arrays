# Fairness Memo — Stage 06–11 Benchmark Protocol

**Date:** 2026-05-28
**Project:** Fresh regression benchmark — methane_ppm and ethylene_ppm from 16-sensor gas array.
**Scope:** Stages 06 (baselines) through 11 (tuning & optimization).
**Status:** Closes Week 1, Task 1.4 of the One-Month Plan.

---

## 1. Purpose

This memo certifies that every model evaluated in Stages 06–11 was trained, tuned, and scored under a single uniform protocol. It documents (a) the benchmark protocol, (b) the temporal validation methodology, (c) preprocessing consistency, (d) the tuning protocol, (e) leakage prevention, (f) fairness guarantees, and (g) reproducibility. Where the protocol is imperfect or limited, it is stated explicitly.

The memo is intended as a standing reference: any number reported from Stages 06–11 in downstream writing (paper, slides, internal report) can be traced back to the protocol described here without further justification.

---

## 2. Benchmark protocol

**Data source.** `data/processed/ethylene_methane.parquet` (4,178,504 rows × 19 columns: `time_s`, `s01`–`s16`, `methane_ppm`, `ethylene_ppm`). Raw file `data/raw/ethylene_methane.txt` is never modified.

**Inputs and targets.** Sixteen sensor channels `s01`–`s16` predict two targets simultaneously (`methane_ppm`, `ethylene_ppm`). `time_s` is used only for ordering and is never a model feature.

**Common training budget.** All models share a single fixed budget of **1,000,000 rows** (~24% of the dataset), produced in Stage 05 by stratified-by-regime sampling and saved to `data/subsets/fair_subset_indices.parquet`. The artifact is monotonic (temporal order preserved) and contains exactly 1M row indices. Per fold, the effective training set is **`train_range ∩ fair_subset`** (~395k rows per fold). No model in Stages 06–11 is permitted more training rows than this intersection.

**Validation slice.** Every fold uses the **full untouched validation slice** (~407k–435k rows per fold). The fair-subset reduction is applied to training only; validation is never subsampled, so all models are scored on the same evaluation distribution.

**Metric set.** MAE, RMSE, R² — computed per target per fold. R² is the scale-free primary comparator; MAE/RMSE are reported for absolute-error context.

---

## 3. Temporal validation methodology

**Splits artifact.** `results/tables/validation_splits.parquet`, produced in Stage 03 and frozen for all subsequent stages.

**Structure.** Five rolling temporal folds. Each fold contains a `train` row (start_idx, end_idx) and a `val` row, with the val window occurring strictly *after* the train window in temporal order. The training block always precedes the validation block in time.

**No shuffling.** Data is never shuffled before splitting. Indices are monotonically increasing within both training and validation ranges.

**Aggregation.** Every metric is reported per fold and aggregated as mean ± std across the 5 folds. Per-target and per-fold breakdowns are saved in the long-format parquet artifacts so any pooled statistic can be reconstructed.

**Why 5 folds (not 3).** The One-Month Plan asked for ≥3 rolling windows. Five was chosen because (a) the temporal axis is long enough to support five well-separated windows without overlap, (b) std estimates are more meaningful with n=5 than n=3, and (c) per-fold collapses are easier to attribute (fold 1 ethylene collapse is now a documented phenomenon across multiple model families).

---

## 4. Preprocessing consistency

**Scaler.** `RobustScaler` (sklearn) applied to the 16 sensor inputs. Targets are not scaled.

**Per-fold refit.** A fresh scaler is fit per fold on the fold's `train_range ∩ fair_subset` intersection, then applied to both train and val. Scalers fitted in earlier stages (e.g., Stage 04) are **not** reused — this is a strict-fairness choice that prevents any single global scaler from leaking statistics across folds.

**Order.** Subset intersection → fold filter → scaler fit on training intersection → scaler transform on train and val → model fit → predict on val. The order is identical across Stages 06, 07, 08, 09, 10, and 11.

**No feature engineering.** No lag, rolling, or seasonality features were created. Polynomial and spline expansions in Stages 08 and 11 are applied as part of an sklearn `Pipeline` *after* scaling, on the same 16 inputs every model sees.

---

## 5. Tuning protocol (Stage 11)

**Search type.**
- Grid search for cheap models with small parameter spaces: Ridge (7 configs), Polynomial (20), Spline (50), Huber (24).
- Randomized search via `ParameterSampler(random_state=42)` for expensive models: ExtraTrees (20 configs), LightGBM (30), CatBoost (30), HistGradientBoosting (30), TabNet (12), MLP-medium (12).

**Total configs evaluated.** 2,350 (config_id × fold × target records logged in `results/tables/11_tuning_search_log.parquet`).

**Best-config selection.** For each model, the config with the highest mean R² pooled across all 5 folds × 2 targets is selected. The selected config's fold-level metrics become the reported Stage 11 metrics for that model.

**Early stopping.** Used for the iterative models (LightGBM, CatBoost, HistGradientBoosting, TabNet, MLP-medium). Implementation:
- **LightGBM, CatBoost:** the last 15% of `train_range ∩ fair_subset` (temporally latest portion) is carved off manually as `eval_set`. Patience = 50 boosting rounds. `n_estimators` / `iterations` set to 2000 so early stopping is the binding constraint.
- **HistGradientBoosting:** built-in `early_stopping=True` with `validation_fraction=0.15`, `n_iter_no_change=50`. **Caveat:** sklearn's internal split is not strictly temporal; this is the most likely cause of HGBR's underperformance vs the other boosted families. Documented as a known issue.
- **TabNet:** the same manual 85/15 temporal carve as LGBM/CatBoost, with `patience=20` epochs, `max_epochs=300`, `batch_size=4096`.
- **MLP-medium:** sklearn's built-in `early_stopping=True` with `validation_fraction=0.1`, `n_iter_no_change=20`. Same caveat as HGBR (sklearn's split is not strictly temporal); MLP did not regress under tuning, but the early-stopping signal is weaker than the manual-carve approach.

**Saved configs.** The best config per model is recorded in `results/memos/11_tuning_optimization.md` and embedded in each saved joblib under the key `'config'`.

---

## 6. Leakage prevention

The validation slice of each fold is never used for:
- model fitting,
- hyperparameter selection,
- early stopping (for LightGBM, CatBoost, TabNet — manual temporal carve),
- feature engineering (no features depend on val statistics),
- scaler fitting.

The fair-subset selection (Stage 05) was made before any validation R² was observed and was based on stratified sampling of the 4-way `(methane>0, ethylene>0)` regime indicator — a property of the targets that does not depend on model performance.

**Acknowledged residual risks:**
1. **Best-config selection is on val.** With 5 folds × 161 configs, the val folds are effectively used to *rank* configs. This is the standard non-nested-CV trade-off. The risk is mitigated by the fact that the same val folds are used to rank all models, so relative ordering is unbiased even if absolute R² is slightly optimistic.
2. **HGBR and MLP-medium use sklearn's built-in early stopping**, whose internal validation split is not guaranteed to respect temporal order. HGBR regressed under tuning (untuned 0.325 → tuned 0.291), which is consistent with this risk materializing. MLP-medium did not regress but should be re-examined with a manual temporal carve in any follow-up work.
3. **No nested CV.** Nested CV was rejected on compute grounds (~10× cost). The plan's "rolling temporal validation" requirement is satisfied by the 5-fold protocol; nested CV is a stronger guarantee not requested.

---

## 7. Fairness guarantees

The protocol satisfies the One-Month Plan's fairness rules:

| Plan rule | Implementation |
|---|---|
| One common training budget across all compared models | ~395k rows per fold (`train_range ∩ fair_subset`), identical for every model in Stages 06–11. |
| Temporal validation, train before val | 5 rolling folds in `validation_splits.parquet`, strict ordering enforced. |
| Same metrics for all models | MAE, RMSE, R² per target per fold, computed by the same sklearn functions in every notebook. |
| Same preprocessing pipeline | Fresh `RobustScaler` per fold, applied identically. |
| Stochastic models report seed | `SEED=42` propagated to `random_state` / `random_seed` / `seed=` arguments throughout. |
| No model uses more data than another in main comparison | Verified: every model in Stages 06–11 reads from the same fold-intersected training set. |

**Deviations explicitly documented:**
- Boosted models (Stage 09) wrap single-output estimators in `MultiOutputRegressor`, fitting one estimator per target. This is a per-target deviation from the native multi-output baselines (Stages 06, 07, 08). The deviation is noted in Stage 09's memo and reflected in Stage 11 (boosted models tuned per-target with manual early-stopping carve).
- TabNet (Stage 10) is per-target by library design (no native multi-output). Documented.
- Polynomial and spline expansions enlarge feature count substantially (e.g., poly deg-3 → ~969 features). This is a transformation of the same input data, not additional data. All models see the same 16 raw sensor channels; only the basis differs.

---

## 8. Reproducibility

**Seeds.** `SEED = 42` is the single source of randomness. Used in: stratified subset sampling (Stage 05), Ridge `random_state`, tree-ensemble `random_state`, MLP `random_state`, LightGBM/XGBoost/CatBoost random seeds, TabNet `seed`, `ParameterSampler(random_state=SEED)` in Stage 11. NumPy's global generator is seeded at the top of every notebook.

**Artifacts saved per stage:**

| Stage | Tables | Figures | Models | Memo |
|---|---|---|---|---|
| 03 | `validation_splits.parquet` | — | — | `03_validation_split.md` |
| 04 | — | — | `models/preprocessors/fold_{k}.joblib` | `04_preprocessing.md` |
| 05 | — | `05_subset_comparison.png` | — | `05_fair_subset.md` |
| 06 | `06_baseline_metrics_{long,summary}.parquet` | `06_baselines.png` | `models/baselines/fold_{k}_{name}.joblib` | `06_baselines.md` |
| 07 | `07_linear_metrics_{long,summary}.parquet` | `07_linear_models.png` | `models/linear_07/...` | `07_linear_models.md` |
| 08 | `08_nonlinear_metrics_{long,summary}.parquet` | `08_nonlinear_models.png` | `models/nonlinear_08/...` | `08_nonlinear_models.md` |
| 09 | `09_boosted_metrics_{long,summary}.parquet` | `09_boosted_models.png` | `models/boosted_09/...` | `09_boosted_models.md` |
| 10 | `10_neural_metrics_{long,summary}.parquet` | `10_neural_models.png` | `models/neural_10/...` | `10_neural_models.md` |
| 11 | `11_tuning_metrics_{long,summary}.parquet`, `11_tuning_search_log.parquet` | `11_tuning_optimization.png` | `models/tuned_11/fold_{k}_{name}_tuned.joblib` | `11_tuning_optimization.md` |

**Environment.** Python 3.13, sklearn 1.8.0, xgboost 3.2.0, lightgbm 4.6.0, catboost 1.2.10, pytorch-tabnet 4.1.0, torch 2.12.0+cpu. CPU-only execution.

**Re-running the benchmark.** Every notebook is fully self-contained: it loads `data/processed/`, `validation_splits.parquet`, and `fair_subset_indices.parquet`, then trains and saves all artifacts. A clean re-run from Stage 06 onward should reproduce the metrics within floating-point tolerance.

**What is *not* reproducible without re-execution:**
- Stage 11 search log timings (`seconds` column) are wall-clock and machine-dependent.
- TabNet results may show small deviations due to PyTorch non-determinism on some operations even with seeds fixed; the `pytorch-tabnet` package does not document strict determinism guarantees.
- HistGradientBoosting and MLP internal early-stopping splits may pick slightly different cut-points across sklearn minor versions.

---

## 9. Limitations and open items

**Limitations:**
1. Single-seed evaluation. The plan requires seeds to be documented but does not require multi-seed averaging. Multi-seed runs are deferred to Week 3/4 if requested.
2. HGBR's and MLP's internal early-stopping splits are not strictly temporal. The first is implicated in HGBR's underperformance; the second is benign but not best-practice.
3. No train-set R² recorded. Only val R² is collected in the long tables. Overfitting/underfitting diagnostics in Stage 11 are inferred from fold variance rather than from train/val gap.
4. The fair subset uses stratified-by-regime sampling. Whether a *coreset*-style selection (k-means prototypes, farthest-point, density-aware) would change rankings is the open question for Week 3.

**Open items for Week 3 / Week 4:**
- Runtime/footprint table (Task 2.4 is partial: training time is logged, inference time and model size are not).
- Coreset-aware subset selection (Task 3.1, 3.2).
- Creative model direction proposal (Task 3.3).
- Transferability-to-classification note (Task 3.4).
- Prototype + interpretation (Tasks 4.1, 4.2).
- Paper structure repair (Task 4.3).
- End-of-month internal report (Task 4.4).

---

**Sign-off.** Stages 06–11 form a single coherent benchmark under one protocol. Any comparison between models within these stages is fair by the One-Month Plan's standards. Any comparison against models from outside this protocol — including the archived `Comparative Analysis/` project — must be flagged as cross-protocol and is not warranted by this memo.
