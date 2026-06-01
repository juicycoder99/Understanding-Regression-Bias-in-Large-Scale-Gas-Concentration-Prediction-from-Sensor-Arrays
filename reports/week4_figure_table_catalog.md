# Manuscript Figure and Table Catalog

**Date:** 2026-05-29
**Project:** Methane and ethylene regression benchmark from 16-sensor gas-array data.
**Purpose:** Publication-ready catalog of every figure and table selected for the manuscript. For each item: exact source file, manuscript number, publication caption, the scientific finding it supports, and main-paper vs appendix placement.
**Scope:** Current-project artifacts only. No legacy `Comparative Analysis/` content. All quantitative values are copied from saved artifacts (verified 2026-05-29); none invented.
**Companion document:** `reports/week4_paper_structure.md` (the section blueprint this catalog populates).

---

## 1. Numbering scheme

- **Figures:** `Figure N` (main paper), `Figure A.N` (appendix).
- **Tables:** `Table N` (main paper), `Table A.N` (appendix).
- Numbering follows narrative order (Section 2 → 6), not artifact filename order.
- Every caption ends with the phase label in brackets: **[baseline] / [tuned] / [coreset] / [prototype] / [setup]**, so a reader never conflates result phases.

---

## 2. Main-paper figures

### Figure 1 — Target distributions and temporal profile
- **Source files:** `results/figures/02_target_distributions.png`, `results/figures/02_target_timeseries.png` (compose as a 2-panel figure).
- **Caption:** *Figure 1. Distribution and temporal profile of the two regression targets. (a) Marginal distributions of methane and ethylene concentration (ppm) over the full 4.18 M-row dataset; methane spans the wider range. (b) Target values across the recording timeline, showing the regime structure exploited by the temporal validation protocol. [setup]*
- **Finding supported:** the targets differ in scale and the signal is temporally non-stationary — the basis for both the methane/ethylene asymmetry and the rolling temporal protocol.
- **Placement:** **Main.**

### Figure 2 — Sensor–target relationships
- **Source files:** `results/figures/02_sensor_target_correlation.png`, `results/figures/02_regime_map.png`.
- **Caption:** *Figure 2. Sensor–target structure. (a) Correlation of each of the 16 sensors (s01–s16) with each target; methane signal concentrates in fewer sensors while ethylene is distributed across more. (b) The four operating regimes defined by (methane>0, ethylene>0), with population shares 32.9% / 22.8% / 24.0% / 20.3%. [setup]*
- **Finding supported:** methane's signal is more localized than ethylene's — the mechanistic root of the per-target difficulty gap seen throughout the results.
- **Placement:** **Main.**

### Figure 3 — Rolling temporal validation layout
- **Source file:** `results/figures/03_fold_layout.png`.
- **Caption:** *Figure 3. Five-fold rolling temporal validation. Each fold's training window strictly precedes its validation window in time; no shuffling is applied. Validation windows range from 407 k to 435 k rows and are never subsampled. [setup]*
- **Finding supported:** the evaluation honors temporal causality, which is what exposes the drift fragility later results document.
- **Placement:** **Main.**

### Figure 4 — Tuned-model performance and the R² ceiling
- **Source file:** `results/figures/11_tuning_optimization.png`.
- **Caption:** *Figure 4. Tuned-model performance under the fair temporal protocol. Per-target R² (mean ± std across 5 folds) for the ten tuned models; the dashed line marks the Ridge linear reference (R² = 0.605). The best model, a regularized cubic spline (spline_tuned), reaches pooled R² = 0.619 (methane 0.661, ethylene 0.578). No model exceeds R² ≈ 0.62. [tuned]*
- **Finding supported:** the headline benchmark result and the first appearance of the ≈ 0.62 ceiling.
- **Placement:** **Main.**

### Figure 5 — Coreset method ranking under fair budget
- **Source file:** `results/figures/13_method_ranking.png`.
- **Caption:** *Figure 5. Coreset method comparison at the 395 k-row per-fold budget. Pooled R² for three frozen reference models (spline, ExtraTrees, CatBoost) under five subset-selection strategies; dashed lines mark each model's Stage-11 reference. Farthest-point sampling (FPS) is best or tied-best for every model and wins 10 of 12 (model × budget) cells overall. [coreset]*
- **Finding supported:** diversity-based selection (FPS) beats representativeness-based (k-means) and random/stratified for predictive quality.
- **Placement:** **Main.**

### Figure 6 — Budget–quality curves
- **Source file:** `results/figures/13_budget_curves.png`.
- **Caption:** *Figure 6. Model quality versus training budget per coreset method, one panel per reference model. Smooth spline rises monotonically with budget, whereas partition-based models peak below full budget (CatBoost at 25 k = 0.5275, ExtraTrees at 60 k = 0.5834) and decline thereafter. [coreset]*
- **Finding supported:** partition models do not need — and are hurt by — the full data budget; a deployment-relevant efficiency result.
- **Placement:** **Main.**

### Figure 7 — Prototype evaluation against the spline baseline
- **Source file:** `results/figures/14_prototype_vs_baselines.png`.
- **Caption:** *Figure 7. Spline-plus-boosting-residual prototype versus baselines. Pooled R² for the six prototype variants; red dashed line is the FPS+spline baseline (0.6221), green dotted line the pre-registered success threshold (0.625). The primary hybrid (spline + CatBoost residual, V1) scores 0.606 — below the spline-only baseline. No variant clears the threshold. [prototype]*
- **Finding supported:** the creative architecture is rejected; adding a residual learner does not help.
- **Placement:** **Main.**

### Figure 8 — Residual structure (why the prototype fails)
- **Source file:** `results/figures/14_residual_structure.png`.
- **Caption:** *Figure 8. Predicted versus true spline residuals on fold-1 validation data, per target. Points show no alignment with the identity line, indicating the residual learner cannot predict the spline's errors. A model that independently reaches R² ≈ 0.51 on the raw targets extracts no usable signal from these residuals. [prototype]*
- **Finding supported:** the mechanistic explanation that the residuals are noise — upgrading the empirical ceiling to a demonstrated information bound.
- **Placement:** **Main.**

---

## 3. Main-paper tables

### Table 1 — Validation protocol windows
- **Source file:** `results/tables/validation_splits.parquet`.
- **Caption:** *Table 1. Five-fold rolling temporal split. Per fold, the training window precedes the validation window in time; validation windows (407 k–435 k rows) are evaluated in full. [setup]*
- **Finding supported:** documents the exact, reproducible temporal protocol shared by every result.
- **Placement:** **Main.**

### Table 2 — Untuned model survey
- **Source files:** `results/tables/06_baseline_metrics_summary.parquet` … `10_neural_metrics_summary.parquet` (consolidated).
- **Caption:** *Table 2. Untuned model survey under the fair temporal protocol (pooled R², mean across folds and targets). Ridge sets the linear reference (0.605); smooth nonlinear models approach it (poly 0.600, spline 0.591) while boosted (best 0.374) and neural (best 0.214) families underperform at library defaults. [baseline]*
- **Finding supported:** the inductive-bias landscape and the motivation for tuning; smooth methods lead from the outset.
- **Placement:** **Main.**

### Table 3 — Tuned-model publication comparison
- **Source files:** `results/tables/11_tuning_metrics_summary.parquet`, cross-checked against `reports/week1_table_audit.md`.
- **Caption:** *Table 3. Tuned-model comparison (per-target R², MAE, RMSE; mean ± std across 5 folds). spline_tuned is the benchmark winner (pooled 0.619; methane 0.661, ethylene 0.578). HistGradientBoosting is the only model to regress under tuning (0.325 → 0.291) owing to a non-temporal internal early-stopping split. [tuned]*
- **Finding supported:** the definitive ranked benchmark and the early-stopping methodological caution.
- **Placement:** **Main.**

### Table 4 — Coreset sensitivity and winners
- **Source files:** `results/tables/13_coreset_sensitivity.parquet`, `results/tables/13_coreset_comparison_summary.parquet`.
- **Caption:** *Table 4. Coreset-selection sensitivity by model and budget. For each (model, budget) cell: best and worst method and the R² spread across the five strategies. Spline is nearly coreset-insensitive (spread 0.004–0.008); CatBoost is the most sensitive (0.027–0.048). FPS is the modal winner. [coreset]*
- **Finding supported:** model-dependent coreset sensitivity; spline is robust, CatBoost needs a good subset.
- **Placement:** **Main.**

### Table 5 — Prototype hypotheses and verdicts
- **Source files:** `results/tables/14_prototype_metrics_summary.parquet`, `results/tables/14_prototype_ablation.parquet`, `results/memos/14_prototype_v1.md`.
- **Caption:** *Table 5. Pre-registered hypotheses H1–H4 for the spline-plus-residual prototype and their outcomes. All four fail: pooled R² 0.606 < 0.625 (H1); residual hurts both targets, ethylene most (H2); fold-1 ethylene worsens by 0.134 (H3); residual coreset swap shifts R² by 0.011 > 0.005 (H4). [prototype]*
- **Finding supported:** the rigor of the rejection — four falsifiable predictions, all refuted.
- **Placement:** **Main.**

### Table 6 — Cross-phase leaderboard
- **Source files:** assembled from `06`–`14` long tables (`results/tables/*_long.parquet`).
- **Caption:** *Table 6. Best result per model across all phases (baseline → tuned → coreset → prototype), pooled R² with per-target breakdown. The benchmark high is FPS-selected spline_tuned at 0.6221; the ≈ 0.62 ceiling holds across every phase. [all phases — labeled per row]*
- **Finding supported:** the single synthesizing artifact showing the ceiling is phase-invariant.
- **Placement:** **Main.**

---

## 4. Appendix figures

### Figure A.1 — Sensor distributions
- **Source file:** `results/figures/02_sensor_boxplot.png`.
- **Caption:** *Figure A.1. Per-sensor value distributions (s01–s16) across the dataset. [setup]*
- **Finding supported:** documents raw sensor ranges motivating RobustScaler.
- **Placement:** **Appendix.**

### Figure A.2 — Baseline-vs-active sensor response
- **Source file:** `results/figures/02_sensor_response_baseline_vs_active.png`.
- **Caption:** *Figure A.2. Sensor response in baseline versus active gas-exposure periods. [setup]*
- **Finding supported:** evidence that sensors carry exposure signal, supporting feature validity.
- **Placement:** **Appendix.**

### Figure A.3 — Sensor–sensor correlation
- **Source file:** `results/figures/02_sensor_sensor_correlation.png`.
- **Caption:** *Figure A.3. Inter-sensor correlation matrix. [setup]*
- **Finding supported:** redundancy among sensors, contextualizing the modest information ceiling.
- **Placement:** **Appendix.**

### Figure A.4 — Sensor time series
- **Source file:** `results/figures/02_sensor_timeseries.png`.
- **Caption:** *Figure A.4. Representative sensor signals over time. [setup]*
- **Finding supported:** visual evidence of temporal drift in the inputs.
- **Placement:** **Appendix.**

### Figure A.5 — Per-fold scaling illustration
- **Source file:** `results/figures/04_fold1_scaling.png`.
- **Caption:** *Figure A.5. Effect of per-fold RobustScaler fitting (fold 1 shown). [setup]*
- **Finding supported:** documents the leakage-safe preprocessing step.
- **Placement:** **Appendix.**

### Figure A.6 — Fair-subset comparison
- **Source file:** `results/figures/05_subset_comparison.png`.
- **Caption:** *Figure A.6. Chronological versus stratified fair-subset construction: regime balance and temporal coverage. [setup]*
- **Finding supported:** the original fair-budget construction underlying the common training budget.
- **Placement:** **Appendix.**

### Figure A.7 — Coreset construction diagnostics
- **Source files:** `results/figures/12_coreset_regime_coverage.png`, `12_coreset_sensor_drift.png`, `12_coreset_dispersion.png`, `12_coreset_temporal_density.png` (2×2 composite).
- **Caption:** *Figure A.7. Coreset construction diagnostics across five methods and four budgets. k-means best preserves sensor means (deviation ≈ 0.001); density-aware sampling most distorts sensor spread (up to 4.6× inflation at 25 k); FPS is the most space-filling. [coreset]*
- **Finding supported:** the representativeness↔diversity trade-off and why density-aware fails — supports the main-paper FPS result.
- **Placement:** **Appendix.**

### Figure A.8 — Coreset sensitivity heatmap
- **Source file:** `results/figures/13_sensitivity_heatmap.png`.
- **Caption:** *Figure A.8. R² spread across coreset methods by model and budget. [coreset]*
- **Finding supported:** compact view that CatBoost is most coreset-sensitive, spline least.
- **Placement:** **Appendix.**

### Figure A.9 — Coreset per-target breakdown
- **Source file:** `results/figures/13_per_target_breakdown.png`.
- **Caption:** *Figure A.9. Per-target R² by coreset method at 395 k budget. [coreset]*
- **Finding supported:** the methane/ethylene gap persists across coreset methods.
- **Placement:** **Appendix.**

### Figure A.10 — Prototype per-fold comparison
- **Source file:** `results/figures/14_per_fold_comparison.png`.
- **Caption:** *Figure A.10. Spline-only versus hybrid prototype per fold and target; fold-1 ethylene degrades under the hybrid. [prototype]*
- **Finding supported:** per-fold evidence behind the H3 failure.
- **Placement:** **Appendix.**

### Figure A.11 — Prototype per-target breakdown
- **Source file:** `results/figures/14_per_target_breakdown.png`.
- **Caption:** *Figure A.11. Per-variant per-target R² for the prototype study. [prototype]*
- **Finding supported:** the residual stage hurts both targets, ethylene most.
- **Placement:** **Appendix.**

### Figures from untuned stages (optional supplementary)
- **Source files:** `06_baselines.png`, `07_linear_models.png`, `08_nonlinear_models.png`, `09_boosted_models.png`, `10_neural_models.png`.
- **Caption (series):** *Figures A.12–A.16. Per-stage untuned diagnostic panels (baselines, linear, nonlinear, boosted, neural): per-model MAE with fold-std error bars and predicted-vs-actual scatters for the best model of each stage. [baseline]*
- **Finding supported:** detailed per-stage evidence consolidated in main-paper Table 2.
- **Placement:** **Appendix** (Table 2 is the main-paper summary; these are the detail).

---

## 5. Appendix tables

### Table A.1 — Cross-protocol leakage audit
- **Source file:** `results/tables/13_cross_protocol_audit.parquet`.
- **Caption:** *Table A.1. Per-fold leakage-clean stratified coreset versus the global stratified reference, per model. All gaps fall within combined fold standard deviation, indicating the earlier global construction was numerically benign. [coreset]*
- **Finding supported:** validates that prior subset construction did not bias results — a methodological assurance.
- **Placement:** **Appendix.**

### Table A.2 — Coreset construction diagnostics (full)
- **Source file:** `results/tables/12_coreset_diagnostics_summary.parquet`.
- **Caption:** *Table A.2. Full coreset construction diagnostics (regime, temporal, sensor, dispersion) for five methods × four budgets, mean ± std across folds. [coreset]*
- **Finding supported:** the numeric backing for Figure A.7.
- **Placement:** **Appendix.**

### Table A.3 — Tuning search log summary
- **Source file:** `results/tables/11_tuning_search_log.parquet`.
- **Caption:** *Table A.3. Hyperparameter search coverage: configurations evaluated per model (2,350 total) and best-config selection by pooled R². [tuned]*
- **Finding supported:** documents tuning breadth, supporting the claim that the ceiling survives exhaustive search.
- **Placement:** **Appendix.**

### Table A.4 — Transfer-strength summary
- **Source file:** `reports/week3_task34_transferability_note.md` (§6 summary table).
- **Caption:** *Table A.4. Expected transferability of regression findings to a classification pipeline, graded by confidence. [—]*
- **Finding supported:** structures the Section 5 transferability discussion.
- **Placement:** **Appendix** (referenced from main-paper Section 5).

---

## 6. Integration directives for the writing pass

So that figures and tables are woven into the narrative rather than appended as external artifacts:

1. **Every main-paper float is called out in prose before it appears**, using the form "Figure N shows…" / "Table N reports…", followed by the *interpretation*, not a restatement of the caption.
2. **Each Section-4 subsection anchors on one or two floats:** 4.1→Table 2; 4.3→Figure 4 + Table 3; 4.4→Figures 5–6 + Table 4; 4.5→Figures 7–8 + Table 5; 4.6→Table 6.
3. **The ceiling thread is carried by three floats in escalating order:** Figure 4 (observed) → Figures 5–6 (survives better selection) → Figure 8 + Table 5 (explained as an information bound). The prose must reference back to the earlier float when reaching the next.
4. **Phase labels in captions are mandatory** and must match the phase stated in the surrounding prose.
5. **No float is orphaned:** every figure/table in this catalog has a designated calling subsection (main) or a single appendix reference from the main text.
6. **All numeric values quoted in prose must match the source artifact** listed here; the catalog is the single reconciliation point between narrative numbers and saved files.

---

## 7. Inventory reconciliation

- **Saved figures:** 29. **Placed:** 29 (8 main as composites/singles, the remainder appendix). 
- **Saved result tables (parquet):** 25. **Rendered as manuscript tables:** 6 main + 4 appendix; the remaining long/parquet files are the underlying data for those rendered tables and the cross-phase leaderboard, not separate manuscript tables.
- **Reports as source material:** `week1_fairness_memo.md`, `week1_table_audit.md`, `week3_task33_creative_direction.md`, `week3_task34_transferability_note.md`, `week4_paper_structure.md` — all current-project, all cited as source, none from the legacy project.

---

**Sign-off.** Every manuscript figure and table is assigned a number, an exact source file, a publication-ready caption, the finding it supports, and a main/appendix placement — drawn solely from current-project artifacts. Integration directives ensure floats are woven into the narrative. This catalog plus `week4_paper_structure.md` are the complete pre-writing blueprint; prose drafting is the next pass. Task 4.4 (closing report) remains separate.
