# 12 - Coreset Strategies (per-fold, leakage-clean)

**Date:** 2026-05-28
**Dataset:** ../data/processed/ethylene_methane.parquet
**Splits artifact:** ../results/tables/validation_splits.parquet
**Subset directory:** ../data/subsets

---

## Summary

- 5 methods x 4 budgets x 5 folds = 100 subset artifacts.
- Construction pool: each fold's `train_range` only. **No validation rows participate in construction.**
- Per-fold `RobustScaler` fit on the fold's train_range, saved as `coreset_fold{k}_scaler.joblib`.
- Diagnostics aggregated across folds and saved long-format for downstream analysis.

## Methods

- **random** - uniform sample without replacement; sorted; no method-induced bias.
- **kmeans** - `MiniBatchKMeans(k=2000)` fit per fold; budget allocated per cluster by largest-remainder; rows picked nearest to centroid.
- **fps** - bucket-based FPS: 1000 temporal buckets per fold, textbook FPS within each bucket up to 395 picks; smaller budgets truncate the precomputed order.
- **density** - BallTree on a 100k anchor subsample of the fold's pool; k=20 NN distance as inverse-density weight; weighted sample without replacement.
- **stratified** - per-fold `(methane>0, ethylene>0)` regime proportions; largest-remainder allocation; sample within each regime.

## Budgets (rows per fold)

- 25,000 (~1.5% of fold pool)
- 60,000 (~3.7% of fold pool)
- 120,000 (~7.3% of fold pool)
- 395,000 (~24% of fold pool, matches Stage 11 implicit per-fold size)

## Diagnostic Summary (mean across 5 folds)

| method | budget | regime_max_abs_dev | sensor_max_abs_dev | nn_dist_mean |
| --- | --- | --- | --- | --- |
| density | 25000 | 0.0296 | 0.9725 | 0.0792 |
| fps | 25000 | 0.0011 | 0.0429 | 0.1135 |
| kmeans | 25000 | 0.0049 | 0.0013 | 0.0299 |
| random | 25000 | 0.0046 | 0.0092 | 0.0702 |
| stratified | 25000 | 0.0 | 0.0059 | 0.0713 |
| density | 60000 | 0.0245 | 0.4791 | 0.0793 |
| fps | 60000 | 0.0008 | 0.0384 | 0.1046 |
| kmeans | 60000 | 0.0046 | 0.001 | 0.0347 |
| random | 60000 | 0.0026 | 0.0063 | 0.0711 |
| stratified | 60000 | 0.0 | 0.0041 | 0.0707 |
| density | 120000 | 0.0234 | 0.2581 | 0.0792 |
| fps | 120000 | 0.0007 | 0.0361 | 0.0968 |
| kmeans | 120000 | 0.0042 | 0.0011 | 0.039 |
| random | 120000 | 0.0019 | 0.0041 | 0.0712 |
| stratified | 120000 | 0.0 | 0.0023 | 0.0711 |
| density | 395000 | 0.0217 | 0.0821 | 0.0786 |
| fps | 395000 | 0.0004 | 0.0273 | 0.0857 |
| kmeans | 395000 | 0.0035 | 0.0012 | 0.0486 |
| random | 395000 | 0.0009 | 0.0024 | 0.0712 |
| stratified | 395000 | 0.0 | 0.0013 | 0.0702 |

## Artifacts

- Subsets: `../data/subsets/coreset_{method}_{budget_label}_fold{k}.parquet` (100 files)
- Per-fold scalers: `../data/subsets/coreset_fold{k}_scaler.joblib` (5 files)
- Diagnostics long: `../results/tables/12_coreset_diagnostics_long.parquet`
- Diagnostics summary: `../results/tables/12_coreset_diagnostics_summary.parquet`
- Figures: `../results/figures/12_coreset_{regime_coverage,temporal_density,dispersion,sensor_drift}.png`

---

## Construction Pool Design Decision and Leakage Analysis

### 1. The original question

For per-fold rolling temporal validation, the coreset construction pool can be drawn from:

- **(A)** the full dataset,
- **(B)** the union of all 5 folds' training ranges,
- **(C)** each fold's training range separately,
- **(D)** the intersection of all 5 folds' training ranges.

Each option fixes which rows are *eligible* to participate in coreset construction and, by extension, which rows can *bias* the construction algorithm's selection of training samples.

### 2. The four options considered

**Option A — full dataset (4.18M rows).** Stage 05's approach. Treats coreset construction as a global preprocessing step decoupled from fold structure.

**Option B — union of all 5 folds' training ranges.** Excludes rows that are *never* in any fold's training set. In this project the 5 training windows overlap, so the union is a substantial fraction of the dataset, but rows that fall in some fold's val slice remain admissible if they are also in another fold's train slice.

**Option C — each fold's training range separately.** Construction is repeated 5 times. For fold *k*, the candidate pool contains only rows in fold *k*'s `train_range`. No row from any val slice (its own or another fold's) can appear in the pool.

**Option D — intersection of all 5 folds' training ranges.** The candidate pool is the set of rows that are training rows for *every* fold. Single global pool, no overlap with any val slice.

### 3. Advantages and disadvantages of each option

| option | pros | cons |
| --- | --- | --- |
| A. full dataset | Cheapest precompute; single global artifact per (method, budget); matches Stage 05. | Val rows participate in construction. Distance-based methods (k-means, FPS, density) can be biased toward val's sensor distribution. |
| B. union of train ranges | Excludes rows that are never train. Single global artifact. | Cross-fold leakage: rows that are val for fold *k* but train for fold *j* (*j != k*) remain in the pool and bias fold *k*'s coreset selection. The five folds' training windows overlap heavily in this project, so this leakage is substantial. |
| C. per-fold | Strongest leakage guarantee: fold *k*'s pool contains no row that is val for any fold. Aligns with the temporal validation philosophy used throughout Stages 06-11. | 5x precompute cost; 5x artifact count (100 instead of 20). Coresets are no longer global; use-site loads per-fold artifacts. |
| D. intersection of train ranges | Single global artifact; no val rows in pool. | The intersection is the earliest temporal slice (rows that are train in every fold, including fold 1 whose training window is earliest). Misses later operating regimes. Misrepresents folds 3-5. |

### 4. Leakage analysis for each option

**A. Full dataset.** Direct leakage path. Every val row contributes to k-means centroids, FPS distances, and density estimates. The training rows selected for similarity to val's sensor distribution will inflate val performance through covariate similarity, not through better generalization.

**B. Union of train ranges.** Indirect leakage path. Suppose row *r* is val for fold 1 and train for fold 2. When constructing fold 1's coreset under Option B, the pool includes *r* (because *r* is in the union of training ranges). The k-means centroids fit on this pool are shaped partly by *r*; the training rows selected as nearest-to-centroid for fold 1 are partly chosen because they are similar to *r*; *r* is then the val row used to evaluate fold 1. This is the same leakage path as Option A, just routed through cross-fold overlap. Because this project's training windows overlap heavily, the leakage volume under Option B is large.

**C. Per-fold.** Construction operates on a pool that is *defined* as fold *k*'s train_range. Every val row of every fold is mechanically excluded from fold *k*'s pool. No leakage path through pool membership exists. Residual leakage paths (such as best-config selection on val performance in Stage 11) are unchanged and acknowledged in the Week 1 fairness memo.

**D. Intersection.** No val rows in pool, hence no construction-pool leakage. However the pool is restricted to the earliest training slice, which biases the coreset toward the operating regime present during fold 1's training period. Folds 3-5 are then trained on a coreset that does not represent their actual operating distribution. This is not leakage but distributional mis-coverage that would degrade model quality without a methodological gain.

### 5. Final decision

**Option C** is adopted. The coreset for fold *k* is constructed using only the rows in fold *k*'s `train_range` as the candidate pool. Precomputations (`MiniBatchKMeans` fit, FPS bucket structure, `BallTree`, density scores) are repeated per fold. The artifact count is 100 (5 methods x 4 budgets x 5 folds).

### 6. Rationale

- **Prevents validation rows from influencing training-row selection.** Mechanical guarantee through pool restriction.
- **Eliminates cross-fold leakage.** The Option B failure mode (a val row of fold *k* still in fold *k*'s pool via cross-fold overlap) cannot occur.
- **Aligns with the temporal validation philosophy used throughout Stages 06-11.** Stages 06-11 enforce strict train-before-val ordering per fold; Stage 12 inherits the same per-fold discipline upstream of model training.
- **Provides the strongest methodological defense for publication and thesis review.** Reviewers asking "did your coreset see the test set" can be answered with a one-line proof: the coreset construction pool was the fold's train_range; the val slice was disjoint by construction.

### 7. Computational cost trade-off

Per-fold construction multiplies the expensive precomputation steps by 5:

| step | global (rejected) | per-fold (adopted) |
| --- | --- | --- |
| K-means fit (k=2000) | ~5-8 min | ~25-40 min |
| FPS bucket precompute | ~10-15 min | ~50 min |
| BallTree + kNN query | ~10-20 min | ~75 min |
| Random + stratified | <1 min | <5 min |
| Diagnostics + figures + memo | ~5 min | ~10-15 min |
| **Wall-clock total** | **~45-60 min** | **~3-3.5 hours** |

The 5x runtime increase is accepted because it buys a clean leakage guarantee that is otherwise unobtainable at this scale.

### 8. Reproducibility note

This decision is recorded as a project-level protocol. All subsequent stages that consume coresets (Task 3.2 / notebook 13, Task 3.3 creative direction, Week 4 prototype) inherit the same per-fold construction policy. Any cross-protocol comparison (e.g., against the Stage 05 1M global stratified artifact) must label the comparison explicitly as cross-protocol and account for the construction-pool difference in interpretation. Per-fold scalers (`coreset_fold{k}_scaler.joblib`) are saved so downstream stages can verify the construction-time scaling used at build time.

---

## Differences from Stage 05

- Stage 05 used the full dataset as construction pool; this stage uses each fold's train_range.
- Stage 05 used full-dataset regime proportions for stratified allocation; this stage uses per-fold pool proportions.
- Stage 05 fit one global `RobustScaler` implicitly through downstream notebooks; this stage saves explicit per-fold scalers fit only on the fold's pool.
- Stage 05 produced one global subset per budget; this stage produces 5 per (method x budget).
- Stage 05's 1M global stratified artifact is **not regenerated** and remains as a separate cross-protocol reference.

## Next step

Notebook `13_coreset_comparison.ipynb` (pending approval): evaluate `spline_tuned`, `extratrees_tuned`, and `catboost_tuned` on each per-fold coreset at each budget. Compare R^2, MAE, RMSE under the same Stage 06-11 protocol.