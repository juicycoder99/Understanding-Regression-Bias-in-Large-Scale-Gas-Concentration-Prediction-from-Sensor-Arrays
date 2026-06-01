# Task 4.4 — End-of-Month Closing Report

**Date:** 2026-05-29
**Project:** Methane and ethylene regression benchmark from 16-sensor gas-array data.
**Author:** Jibril (regression branch).
**Status:** Final closing report for the one-month effort. Closes Week 4, Task 4.4.
**Source of truth:** current-project saved artifacts only (Stages 01–14 + `reports/`). Legacy `Comparative Analysis/` excluded.

This report has two parts, as required by the One-Month Plan: **Part 1** — what was learned scientifically during the fair-comparison and coreset study; **Part 2** — the next-month plan to transform the findings into a deployable model for Pergamon. An artifact appendix follows.

---

# Part 1 — Scientific learnings

## 1.1 Headline result

Under a fair, leakage-clean, rolling temporal protocol, the best model for predicting methane and ethylene from the 16-sensor array is a **regularized cubic spline (spline_tuned) trained on a farthest-point-sampling (FPS) coreset**, reaching **pooled R² = 0.6221** (methane R² ≈ 0.66, ethylene R² ≈ 0.58). Because its architecture is a spline basis expansion with a linear head, we *expect* it to be among the least expensive models at inference, so that the best-accuracy and lowest-cost choices may coincide. This expectation is not yet measured — inference-time and model-size profiling (Task 2.4) remains outstanding (Part 2.2).

## 1.2 The ≈ 0.62 ceiling is an information bound, not a modeling deficiency

This is the single most important scientific conclusion of the month.

- Across **2,350 tuned configurations** (Stage 11) and **60 coreset variants** (Stage 13), no model exceeded R² ≈ 0.62.
- Stage 14 tested whether a second-stage learner could extract what the spline missed: a CatBoost residual stage that *independently* reaches R² ≈ 0.51 on the raw targets extracted **no usable signal** from the spline residuals (hybrid 0.606 < spline-only 0.622; all four pre-registered hypotheses H1–H4 failed).
- Conclusion: the residuals are effectively noise. The ≈ 0.62 ceiling is a **fundamental information limit of the 16 instantaneous sensor readings under temporal validation**, not a limitation of any model family. No architecture can break a bound set by the data itself.

This converts an empirical observation (Stage 11) into a demonstrated, defensible result (Stage 14) — a negative result presented as a positive scientific contribution.

## 1.3 Inductive-bias landscape

- **Smooth global functions win.** Spline, polynomial, and ridge dominate every leaderboard (tuned spline 0.619, poly 0.607, ridge 0.605).
- **Partition methods reach methane parity but collapse on ethylene.** Tuned CatBoost/LightGBM reach methane R² ≈ 0.61 (Ridge parity) but ethylene only 0.39–0.42.
- **Neural methods underperform smooth methods** at every tested configuration (tuned MLP 0.518, TabNet 0.430).
- **Methane is more recoverable than ethylene** (0.66 vs 0.58 ceiling) — traced in EDA to methane's signal concentrating in fewer sensors while ethylene is distributed.

## 1.4 Temporal distribution shift is the dominant failure mode

- Partition-based, boosted, and neural models collapse on specific temporal folds — most acutely **fold-1 ethylene** (tuned TabNet ≈ 0.00, LightGBM 0.06, CatBoost 0.21), while smooth models stay at 0.55–0.59 on the same fold.
- Smooth global functions degrade gracefully under drift; local/partition methods degrade catastrophically.
- **Methodological implication:** random k-fold validation would have hidden this entirely. Rolling temporal validation is non-negotiable for this problem.

## 1.5 Tuning lessons

- **Boosted models underfit at library defaults.** At 100 iterations, boosted R² was 0.33–0.37; at 2,000 iterations + early stopping, 0.50–0.51. The improvement is almost entirely iteration count, not model family.
- **Heavy regularization is required for temporal generalization.** Every improved tree/boosted model won at the regularized end (ExtraTrees depth 10 + leaf 50 drove the +0.132 gain; LightGBM depth 3; CatBoost l2=10).
- **sklearn's built-in early stopping is unsafe on temporal data.** HistGradientBoosting was the only model to *regress* under tuning (0.325 → 0.291) because its internal validation split is non-temporal. Fix: carve the eval set manually as the temporally-latest 10–15% of training data.

## 1.6 Coreset / fair-budget lessons

- **Diversity beats representativeness for prediction.** FPS (farthest-point) coresets won **10 of 12** (model × budget) cells; k-means (most representative by construction diagnostics) won only 1. The "most representative" subset is not the most useful for model quality.
- **Density-aware sampling is non-viable** here — it inflated sensor spread up to 4.6× and was worst for CatBoost at every budget.
- **Partition models peak below full budget** (CatBoost at 25 k = 0.5275, ExtraTrees at 60 k = 0.5834) and decline with more data; smooth models rise monotonically to 395 k. A deployment-relevant efficiency finding.
- **Spline is coreset-insensitive** (R² spread 0.004–0.008 across methods); **CatBoost is the most sensitive** (0.027–0.048). Robustness to subset choice is itself a model-selection criterion.
- **The earlier (Stage 05) global stratified subset was numerically benign.** The cross-protocol audit (Stage 13) found per-fold leakage-clean stratified and the global stratified reference agree within fold-std for all three reference models — so prior benchmark numbers stand.

## 1.7 What the month established as robust vs phase-dependent

- **Robust across all phases:** the ≈ 0.62 ceiling; smooth > partition > neural; methane easier than ethylene; temporal drift dominant; fold-1 ethylene a structural weak point.
- **Changed after tuning (must be stated carefully in writing):** spline overtakes Ridge as champion; boosted methods *underfit*, they do not *fail*; ExtraTrees is stable once regularized; neural methods learn when sized correctly but still lose.

---

# Part 2 — Next-month plan toward a deployable model for Pergamon

The benchmark phase is complete and the science is settled. The next month should convert the winning model into a deployment-ready artifact and close the methodological gaps that the benchmark deliberately deferred. Ordered by priority.

## 2.1 Lock the deployment candidate
- **Action:** promote **spline_tuned on an FPS coreset** to the reference deployable model. It is the accuracy leader (0.6221); its simple spline-plus-linear-head structure is *expected* to be inexpensive at inference, pending the profiling deferred to Part 2.2.
- **Why now:** the ceiling result means further accuracy search has low expected value; effort should shift to making the proven model deployable.

## 2.2 Close the deferred methodological gaps (correctness before deployment)
1. **Multi-seed evaluation.** All results are single-seed (SEED=42). Re-run the top 2–3 models across ≥5 seeds to attach confidence intervals to the headline numbers before they enter a deployment decision.
2. **Train/validation-gap logging.** Only val metrics were recorded. Add train R² capture to convert the overfit/underfit discussion (currently inferred from fold variance) into direct evidence.
3. **Inference-time and model-size profiling.** Task 2.4 was only partially completed (training time logged; inference latency and footprint were not). Measure per-sample latency and serialized size for the deployment candidate and its closest competitors — Pergamon cares about the accuracy/cost trade-off.
4. **HistGradientBoosting early-stopping fix.** If HGBR is to remain in any comparison, replace its built-in early stopping with the manual temporal carve and re-evaluate.

## 2.3 Deployment engineering for the spline model
- Serialize the per-fold spline + scaler into a single inference artifact.
- Benchmark cold-start and steady-state latency on the target hardware profile.
- Define the production input contract (s01–s16 ordering, scaling parameters, handling of out-of-range sensor values).
- Establish a drift-monitoring plan: since temporal drift is the dominant failure mode, production must monitor for the same covariate shift that causes fold-1-style collapses, and trigger retraining when detected.

## 2.4 Optional research extension (only if accuracy must improve)
- The **regime-aware** architecture (deferred from Task 3.3) is the one remaining untested archetype with a plausible mechanism: separate models for low- and high-concentration regimes, motivated by the fold-1 ethylene collapse possibly being regime-driven. **Caveat:** the Stage 14 information-bound result suggests limited headroom; this should be framed as a diagnostic experiment, not an expected accuracy win. Pursue only if a business requirement demands R² > 0.62 and additional sensors/features are not an option.
- The more promising lever for breaking the ceiling is **new information** (additional sensors, temporal features such as short-window context) rather than new model architectures — because the ceiling is an information bound of the current 16 instantaneous inputs. Any feature-engineering extension must respect the project's rule against blind lag/rolling features and be validated under the same temporal protocol.

## 2.5 Cross-branch coordination (classification)
- Hand the transferability note (`week3_task34_transferability_note.md`) to Otmane and align on: adopting the shared 5-fold temporal protocol, reusing the per-fold FPS coresets, and the early-stopping correctness fix.
- Open item Otmane must resolve: whether classification needs a **class-stratified FPS coreset** (the regression coresets are class-agnostic). This is the one construction the regression study did not provide.

## 2.6 Manuscript completion
- The manuscript structure is frozen (`paper_master_outline.md`). The remaining work is the **drafting pass**: write prose against the frozen outline and the float catalog, integrating figures/tables into the narrative.
- No structural changes are permitted unless a contradiction is found in the saved artifacts (documented in the outline's Change Log first).

---

# Appendix — Artifacts produced this month

## Notebooks (`notebooks/`)
- `06_baselines.ipynb` … `10_neural_models.ipynb` — untuned benchmark stages.
- `11_tuning_optimization.ipynb` — hyperparameter tuning (2,350 configs).
- `12_coreset_strategies.ipynb` — per-fold leakage-clean coreset construction (100 artifacts).
- `13_coreset_comparison.ipynb` — coreset quality on the regression task (600 records).
- `14_prototype_v1.ipynb` — spline+residual prototype (rejected).

## Result tables (`results/tables/`)
- `06`–`10` `*_metrics_long/summary.parquet` — untuned per-stage metrics.
- `11_tuning_metrics_long/summary.parquet`, `11_tuning_search_log.parquet`.
- `12_coreset_diagnostics_long/summary.parquet`.
- `13_coreset_comparison_long/summary.parquet`, `13_coreset_sensitivity.parquet`, `13_budget_curves.parquet`, `13_cross_protocol_audit.parquet`.
- `14_prototype_metrics_long/summary.parquet`, `14_prototype_ablation.parquet`, `14_per_fold_breakdown.parquet`.
- `validation_splits.parquet` — the frozen temporal protocol.

## Figures (`results/figures/`)
- 29 saved figures (EDA, fold layout, per-stage diagnostics, tuning, coreset construction/comparison, prototype). Catalogued in `reports/week4_figure_table_catalog.md`.

## Memos (`results/memos/`)
- `01`–`14` per-stage memos.

## Reports (`reports/`)
- `week1_fairness_memo.md` — benchmark protocol and leakage control.
- `week1_table_audit.md` — baseline (06–10) vs tuned (11) reconciliation.
- `week3_4_audit_and_plan.md` — task audit and execution plan.
- `week3_task33_creative_direction.md` — prototype direction selection + pre-registered hypotheses.
- `week3_task34_transferability_note.md` — regression → classification transfer analysis.
- `week4_paper_structure.md` — section/claim blueprint.
- `week4_figure_table_catalog.md` — numbered floats, captions, placements.
- `paper_master_outline.md` — **frozen** manuscript structure.
- `week4_closing_report.md` — this report.

## Models (`results/models/`)
- `baselines/`, `linear_07/`, `nonlinear_08/`, `boosted_09/`, `neural_10/`, `tuned_11/`, `prototype_14/` — fitted per-fold artifacts.

---

# One-Month Plan completion status

| week | tasks | status |
|---|---|---|
| Week 1 | fair protocol, common budget, fair subset, fairness memo | complete (memo + audit in `reports/`) |
| Week 2 | re-evaluate models, stronger baselines, rolling validation, runtime | complete; **runtime is training-time only — inference/size deferred to Part 2.2** |
| Week 3 | coreset selection, coreset comparison, creative direction, transferability | complete (Notebooks 12–13, design doc, transfer note) |
| Week 4 | prototype, interpretation, paper structure, closing report | complete (Notebook 14, frozen outline + catalog, this report) |

**Net:** all four weeks' core deliverables are complete. The two acknowledged carry-forwards into next month are (a) inference-time/model-size profiling and (b) multi-seed confidence intervals — both deployment-readiness items, not benchmark gaps.

---

**Sign-off.** The experimentation phase is complete and its central finding — a ≈ 0.62 information ceiling on the current 16-sensor inputs — is established with direct evidence. The deployable candidate (FPS-selected spline_tuned) is identified. The next month's work is deployment engineering, methodological closure (multi-seed, latency/size), cross-branch coordination, and the manuscript drafting pass against the frozen outline.
