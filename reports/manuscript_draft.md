# A Fair, Temporally-Validated Benchmark for Gas-Concentration Regression from Sensor Arrays, with a Diagnostic Information-Ceiling Analysis

**Manuscript draft — v1 (2026-05-29).** Drafted against the frozen structure `reports/paper_master_outline.md`. Section ordering, figure numbering, table numbering, and narrative spine follow the frozen outline. All quantitative values are taken from saved artifacts (`results/tables/`, `results/memos/`); none are invented. Floats are called out by their frozen numbers. The legacy `Comparative Analysis/` project is not referenced.

---

## Abstract

We benchmark regression models for predicting methane and ethylene concentration from a 16-channel gas-sensor array, under a deliberately fair protocol: a single common training budget, five rolling temporal validation folds in which training always precedes validation, per-fold preprocessing, and leakage-controlled early stopping. Across twenty model variants and 2,350 tuned configurations, a regularized cubic spline is the strongest model (pooled R² = 0.619), narrowly exceeding a Ridge linear reference (0.605); boosted and neural families underperform smooth global functions at every setting. We then ask whether the choice of training subset, rather than the model, limits accuracy. Comparing five coreset-construction strategies across four budgets, farthest-point (diversity-maximizing) sampling is best or tied-best for every model and yields the benchmark high of R² = 0.6221, while representativeness-maximizing k-means does not. Finally we test a creative two-stage architecture — a spline backbone with a boosted residual learner — and reject it: it fails four pre-registered hypotheses and performs below the spline alone, because the spline residuals carry no learnable structure. We conclude that the observed R² ≈ 0.62 is a fundamental **information ceiling** of the 16 instantaneous sensor inputs under temporal validation, not a modeling deficiency. The recommended model is also structurally simple — a spline basis with a linear head — which we expect to translate into low inference cost, a property still to be confirmed by direct profiling. We close with the methodological lessons that transfer to a classification version of the task.

**Keywords:** gas sensor array, regression, temporal validation, coreset selection, methane, ethylene, model benchmarking, information ceiling.

---

## 1. Introduction

Detecting and quantifying gases from low-cost sensor arrays is central to deployable environmental and industrial monitoring. The practical difficulty is not fitting a model to historical data but predicting reliably on *future* data, where sensor behavior drifts away from the training distribution. A benchmark that shuffles its data hides exactly this difficulty and reports optimistic numbers that do not survive deployment.

This paper makes three contributions. First, we establish a **fair, temporally-validated, budget-aware benchmark** for predicting methane and ethylene concentration from 16 sensors, in which every model is trained on the same data volume and evaluated on strictly future time windows. Second, we use the benchmark as a **diagnostic instrument**: by separating the effects of model family, hyperparameter tuning, and training-subset selection, we localize where accuracy is gained and where it is bounded. Third, we test and reject a creative two-stage model, and in doing so demonstrate that the benchmark's accuracy ceiling is an **information limit of the inputs**, not a limitation of any model — a result with direct consequences for deployment and for a future classification version of the task.

The paper is organized so that each stage poses the question the next answers: the data and protocol (Sections 2–3) define the contract; the baseline and tuned benchmarks (Sections 4.1–4.3) establish the ceiling; the coreset study (Section 4.4) tests whether fairer data selection breaks it; the prototype (Section 4.5) tests whether a richer architecture breaks it; and the synthesis, transferability outlook, and conclusion (Sections 4.6–6) state what is robust and what it means for deployment.

---

## 2. Data and Problem Setup

### 2.1 Dataset and targets

The dataset comprises 4,178,504 timestamped records, each with sixteen sensor channels (s01–s16) and two continuous targets, methane and ethylene concentration in parts per million. The recording column `time_s` is used solely to order the data for temporal validation and is never used as a model feature. Figure 1 shows the marginal distributions and temporal profiles of the two targets; methane spans the wider concentration range, which later amplifies its error contribution relative to ethylene.

> **Figure 1.** *Distribution and temporal profile of the two regression targets. (a) Marginal distributions of methane and ethylene concentration (ppm) over the full 4.18 M-row dataset; methane spans the wider range. (b) Target values across the recording timeline, showing the regime structure exploited by the temporal validation protocol.* [setup]
> Source: `results/figures/02_target_distributions.png`, `results/figures/02_target_timeseries.png`.

### 2.2 Sensor–target structure

The sixteen sensors are not equally informative about the two gases. Figure 2(a) shows that the methane signal concentrates in a smaller subset of sensors, whereas the ethylene signal is distributed more broadly across the array. This asymmetry recurs throughout the results: methane proves consistently more recoverable than ethylene, and penalties that prune features (such as L1 regularization in the linear stage) damage ethylene prediction more than methane.

> **Figure 2.** *Sensor–target structure. (a) Correlation of each of the 16 sensors (s01–s16) with each target; methane signal concentrates in fewer sensors while ethylene is distributed across more. (b) The four operating regimes defined by (methane>0, ethylene>0), with population shares 32.9% / 22.8% / 24.0% / 20.3%.* [setup]
> Source: `results/figures/02_sensor_target_correlation.png`, `results/figures/02_regime_map.png`.

### 2.3 Operating regimes and temporal non-stationarity

The data contains four operating regimes defined by the presence or absence of each gas (Figure 2b), with population shares of 32.9%, 22.8%, 24.0%, and 20.3%. Because these regimes occur in temporally contiguous blocks, the feature distribution seen during any training window differs systematically from that of a later validation window. This non-stationarity is the central challenge the protocol in Section 3 is designed to measure honestly, and the mechanism behind the temporal-drift failures documented in Section 4.

This temporal structure leaves the next question open: given non-stationary data, how is the comparison made fair and leakage-free?

---

## 3. Methodology

### 3.1 Rolling temporal validation

We evaluate every model with five rolling temporal folds. In each fold the training window strictly precedes the validation window in time, and the data is never shuffled. Table 1 reports the fold boundaries; validation windows range from 407,362 to 435,098 rows and are always evaluated in full (never subsampled), so all models are scored on the same evaluation distribution.

> **Table 1.** *Five-fold rolling temporal split. Per fold, the training window precedes the validation window in time; validation windows (407 k–435 k rows) are evaluated in full.* [setup]
> Source: `results/tables/validation_splits.parquet`.

| fold | train rows | val rows |
|---|---|---|
| 1 | 1,664,421 | 407,367 |
| 2 | 1,634,650 | 419,449 |
| 3 | 1,641,557 | 407,362 |
| 4 | 1,641,541 | 431,972 |
| 5 | 1,671,616 | 435,098 |

Figure 3 illustrates the rolling layout.

> **Figure 3.** *Five-fold rolling temporal validation. Each fold's training window strictly precedes its validation window in time; no shuffling is applied. Validation windows range from 407 k to 435 k rows and are never subsampled.* [setup]
> Source: `results/figures/03_fold_layout.png`.

### 3.2 Common training budget

To make model comparisons fair, every model is trained on the same data volume. A fixed common subset (the "fair subset") of one million rows is drawn with the regime distribution preserved; intersected with each fold's training window it yields approximately 395,000 training rows per fold. No model is permitted more training data than this in the main comparison. The budget is applied to training only; validation windows remain full.

### 3.3 Per-fold preprocessing

Within each fold a fresh `RobustScaler` is fit on that fold's training data alone and then applied to both training and validation features. Scalers are never shared across folds, so no validation statistics leak into training through preprocessing.

### 3.4 Leakage control and early stopping

The validation window of a fold is never used for fitting, hyperparameter selection, feature construction, or scaler fitting. For iterative models that require early stopping (boosted trees, neural networks), the stopping signal is taken from a manual carve of the temporally-latest 10–15% of the training data, never from the validation window. As Section 4.3 shows, models that instead rely on a library's built-in non-temporal early-stopping split are penalized.

### 3.5 Reproducibility

A single random seed (42) governs all stochastic operations, and every reported result is written to a saved artifact (a metrics table, a figure, or a fitted model). The protocol described here is the contract inherited unchanged by every result in Section 4.

With the contract fixed, we ask what models can actually achieve under it.

---

## 4. Experimental Results and Discussion

### 4.1 Fair benchmark protocol and baselines

We first establish the floor. A Ridge linear model sets the linear reference at pooled R² = 0.605 (methane 0.631, ethylene 0.579), and an ordinary least-squares model matches it; a mean predictor confirms the floor at R² = −0.227. These baselines, reported alongside the full untuned survey in Table 2, anchor the scale against which every later result is read. The question they raise is whether any model can beat the linear floor.

### 4.2 Untuned model survey

We surveyed four model families at sensible library defaults under the fair protocol. Table 2 reports pooled R² (mean across folds and targets). Within the linear/statistical family no model beats Ridge; robust Huber regression is the best at 0.564. Among nonlinear methods, smooth global functions approach the ceiling (polynomial 0.600, spline 0.591) while tree and neighbour methods collapse under temporal drift (random forest 0.202, KNN 0.119, decision tree −0.018). Boosted models underfit badly at 100 iterations (best 0.374), and neural models underperform every smooth method (TabNet 0.214, MLP 0.069). Detailed per-stage diagnostic panels are provided in the appendix (Figures A.12–A.16).

> **Table 2.** *Untuned model survey under the fair temporal protocol (pooled R², mean across folds and targets). Ridge sets the linear reference (0.605); smooth nonlinear models approach it (poly 0.600, spline 0.591) while boosted (best 0.374) and neural (best 0.214) families underperform at library defaults.* [baseline]
> Source: `results/tables/06_baseline_metrics_summary.parquet` … `10_neural_metrics_summary.parquet`.

| family | best model | pooled R² |
|---|---|---|
| baseline (linear) | ridge | 0.605 |
| linear/statistical | huber | 0.564 |
| nonlinear (smooth) | poly | 0.600 |
| nonlinear (smooth) | spline | 0.591 |
| nonlinear (partition) | extratrees | 0.434 |
| boosted | gbr | 0.374 |
| neural | tabnet | 0.214 |
| floor | mean | −0.227 |

The survey leaves one question: is the collapse of the partition, boosted, and neural families a fundamental limitation, or merely underfitting that tuning can repair?

### 4.3 Tuned-model evaluation

We tuned ten models across 2,350 configurations under the unchanged protocol. Figure 4 and Table 3 report the outcome. The benchmark winner is a regularized cubic spline (spline_tuned) at pooled R² = 0.619 (methane 0.661, ethylene 0.578), narrowly above the Ridge reference. Tuning's gains trace to three mechanisms: iteration count (boosted models rose from ≈0.33 at 100 iterations to ≈0.50 at 2,000 with early stopping), regularization strength (every improved tree/boosted model won at the regularized end of its search), and the optimizer regime for neural networks. One model, HistGradientBoosting, *regressed* under tuning (0.325 → 0.291) because its built-in early stopping uses a non-temporal internal split — a concrete caution against library-default early stopping on temporal data. Despite the breadth of the search, no model exceeded R² ≈ 0.62. This is the **first appearance of the ceiling**, here as an empirical observation.

> **Figure 4.** *Tuned-model performance under the fair temporal protocol. Per-target R² (mean ± std across 5 folds) for the ten tuned models; the dashed line marks the Ridge linear reference (R² = 0.605). The best model, a regularized cubic spline (spline_tuned), reaches pooled R² = 0.619 (methane 0.661, ethylene 0.578). No model exceeds R² ≈ 0.62.* [tuned]
> Source: `results/figures/11_tuning_optimization.png`.

> **Table 3.** *Tuned-model comparison (per-target R²; mean across 5 folds). spline_tuned is the benchmark winner (pooled 0.619; methane 0.661, ethylene 0.578). HistGradientBoosting is the only model to regress under tuning (0.325 → 0.291) owing to a non-temporal internal early-stopping split.* [tuned]
> Source: `results/tables/11_tuning_metrics_summary.parquet`; cross-checked against `reports/week1_table_audit.md`. Search coverage: Table A.3.

| model | methane R² | ethylene R² | pooled R² |
|---|---|---|---|
| spline_tuned | 0.661 | 0.578 | 0.619 |
| poly_tuned | 0.654 | 0.560 | 0.607 |
| ridge_tuned | 0.631 | 0.579 | 0.605 |
| huber_tuned | 0.594 | 0.556 | 0.575 |
| extratrees_tuned | 0.626 | 0.509 | 0.567 |
| mlp_tuned | 0.536 | 0.501 | 0.518 |
| catboost_tuned | 0.607 | 0.421 | 0.514 |
| lgbm_tuned | 0.605 | 0.393 | 0.499 |
| tabnet_tuned | 0.587 | 0.273 | 0.430 |
| hgbr_tuned | 0.284 | 0.298 | 0.291 |

The ceiling now poses its own question: is R² ≈ 0.62 a property of the data, or an artifact of which 395,000 rows each model happened to train on?

### 4.4 Coreset construction and fair-budget comparison

To separate the effect of *which* training rows are used from the effect of the model, we constructed coresets — principled reduced training sets — under five strategies (random, k-means prototypes, farthest-point sampling, density-aware, and regime-stratified) at four budgets (25 k, 60 k, 120 k, 395 k rows per fold). All coresets are built per fold from that fold's training window only, so no validation row participates in construction; 100 such artifacts were produced and validated. Construction diagnostics (appendix Figure A.7, Table A.2) show a clear trade-off: k-means is the most representative (sensor-mean deviation ≈ 0.001) while farthest-point sampling is the most space-filling, and density-aware sampling badly distorts the sensor distribution (up to 4.6× spread inflation at the smallest budget).

The decisive question is which trade-off helps prediction. Figure 5 and Table 4 answer it: **farthest-point sampling (FPS) is best or tied-best for every reference model and wins 10 of 12 (model × budget) cells**, whereas the most "representative" k-means coreset wins only one. The benchmark high is reached by FPS-selected spline at the 395 k budget, R² = 0.6221 (methane 0.659, ethylene 0.585). Figure 6 adds a deployment-relevant finding: partition-based models peak *below* full budget (CatBoost at 25 k = 0.5275, ExtraTrees at 60 k = 0.5834) and decline with more data, while the smooth spline rises monotonically. Table 4 also shows that spline is nearly insensitive to coreset choice (R² spread 0.004–0.008) whereas CatBoost is the most sensitive (0.027–0.048) — robustness to subset selection is itself a model-selection criterion. A cross-protocol audit (appendix Table A.1) confirms the earlier global subset construction was numerically benign, so prior benchmark numbers stand.

> **Figure 5.** *Coreset method comparison at the 395 k-row per-fold budget. Pooled R² for three frozen reference models (spline, ExtraTrees, CatBoost) under five subset-selection strategies; dashed lines mark each model's Stage-11 reference. Farthest-point sampling (FPS) is best or tied-best for every model and wins 10 of 12 (model × budget) cells overall.* [coreset]
> Source: `results/figures/13_method_ranking.png`.

> **Figure 6.** *Model quality versus training budget per coreset method, one panel per reference model. Smooth spline rises monotonically with budget, whereas partition-based models peak below full budget (CatBoost at 25 k = 0.5275, ExtraTrees at 60 k = 0.5834) and decline thereafter.* [coreset]
> Source: `results/figures/13_budget_curves.png`.

> **Table 4.** *Coreset-selection sensitivity by model and budget. For each (model, budget) cell: best and worst method and the R² spread across the five strategies. Spline is nearly coreset-insensitive (spread 0.004–0.008); CatBoost is the most sensitive (0.027–0.048). FPS is the modal winner.* [coreset]
> Source: `results/tables/13_coreset_sensitivity.parquet`, `results/tables/13_coreset_comparison_summary.parquet`.

| model | R² spread across methods | most coreset-sensitive? |
|---|---|---|
| spline_tuned | 0.004 – 0.008 | least sensitive |
| extratrees_tuned | 0.007 – 0.012 | moderate |
| catboost_tuned | 0.027 – 0.048 | most sensitive |

Fairer data selection moved the ceiling by only +0.003 (0.619 → 0.6221). The ceiling survives better data. The remaining question: can a richer model architecture break it?

### 4.5 Creative prototype and its rejection

From four candidate architectures we selected, on the prior evidence, a two-stage hybrid: a spline backbone (the proven winner) plus a boosted residual learner trained on an FPS coreset, intended to capture whatever local structure the smooth backbone misses. Four hypotheses were pre-registered before running it (H1: pooled R² ≥ 0.625; H2: ethylene gains more than methane; H3: fold-1 ethylene improves by ≥ 0.05; H4: the result is insensitive to the residual coreset).

The architecture is rejected. Figure 7 and Table 5 show that **all four hypotheses fail**: the primary hybrid scores 0.606 — *below* the spline-only baseline of 0.6221 — and the only variant that matches the baseline (an ExtraTrees-residual variant at 0.624) does so within noise. Figure 8 explains why: the predicted residuals show no alignment with the true residuals, i.e., a learner that independently reaches R² ≈ 0.51 on the raw targets extracts no usable signal from the spline residuals. This **upgrades the ceiling from an empirical observation to a demonstrated information bound**: the residuals are effectively noise, so no second-stage model can recover further accuracy from the same 16 inputs.

> **Figure 7.** *Spline-plus-boosting-residual prototype versus baselines. Pooled R² for the six prototype variants; red dashed line is the FPS+spline baseline (0.6221), green dotted line the pre-registered success threshold (0.625). The primary hybrid (spline + CatBoost residual, V1) scores 0.606 — below the spline-only baseline. No variant clears the threshold.* [prototype]
> Source: `results/figures/14_prototype_vs_baselines.png`.

> **Figure 8.** *Predicted versus true spline residuals on fold-1 validation data, per target. Points show no alignment with the identity line, indicating the residual learner cannot predict the spline's errors. A model that independently reaches R² ≈ 0.51 on the raw targets extracts no usable signal from these residuals.* [prototype]
> Source: `results/figures/14_residual_structure.png`.

> **Table 5.** *Pre-registered hypotheses H1–H4 for the spline-plus-residual prototype and their outcomes. All four fail.* [prototype]
> Source: `results/tables/14_prototype_metrics_summary.parquet`, `results/tables/14_prototype_ablation.parquet`, `results/memos/14_prototype_v1.md`.

| hypothesis | threshold | observed | verdict |
|---|---|---|---|
| H1 pooled R² ≥ 0.625 | 0.625 | 0.606 | FAIL |
| H2 ethylene gain > methane gain | eth > meth | meth −0.005, eth −0.027 | FAIL |
| H3 fold-1 ethylene gain ≥ 0.05 | +0.05 | −0.134 | FAIL |
| H4 \|residual-coreset swap\| ≤ 0.005 | 0.005 | 0.011 | FAIL |

With both levers exhausted — better data selection (4.4) and a richer architecture (4.5) — we can state what is robust and what it means for deployment.

### 4.6 Synthesis and discussion

Table 6 assembles the best result per model across all phases. Several findings are stable across baseline, tuned, coreset, and prototype phases: the R² ≈ 0.62 ceiling; the ordering smooth > partition > neural; methane being more recoverable than ethylene; temporal drift as the dominant failure mode; and fold-1 ethylene as a structural weak point (where tuned partition and neural models fall to R² ≈ 0.00–0.21 while smooth models hold 0.55–0.59). Two findings change after tuning and must be stated carefully: the spline overtakes Ridge as champion, and the boosted methods *underfit* rather than *fail* — at sufficient iterations they reach methane parity with Ridge, though never ethylene parity. For deployment the practical reading is encouraging but partly unmeasured: the most accurate model (the spline) is also the structurally simplest — a basis expansion with a linear head — which we *expect* to make it inexpensive at inference, though direct latency and model-size measurements remain outstanding (Section 6.3); and partition models demonstrably need less *training* data than expected (Figure 6).

> **Table 6.** *Best result per model across all phases (baseline → tuned → coreset → prototype), pooled R² with per-target breakdown. The benchmark high is FPS-selected spline_tuned at 0.6221; the ≈ 0.62 ceiling holds across every phase.* [all phases — labeled per row]
> Source: assembled from `results/tables/06`–`14` `*_long.parquet`.

| model | best phase | methane R² | ethylene R² | pooled R² |
|---|---|---|---|---|
| spline | coreset (FPS 395k) | 0.659 | 0.585 | 0.6221 |
| poly | tuned | 0.654 | 0.560 | 0.607 |
| ridge | baseline/tuned | 0.631 | 0.579 | 0.605 |
| huber | tuned | 0.594 | 0.556 | 0.575 |
| extratrees | coreset (FPS 60k) | — | — | 0.5834 |
| catboost | coreset (FPS 25k) | — | — | 0.5275 |
| mlp | tuned | 0.536 | 0.501 | 0.518 |
| tabnet | tuned | 0.587 | 0.273 | 0.430 |

The robustness of these data-driven findings raises a final question: do they transfer beyond regression?

---

## 5. Transferability Outlook

### 5.1 Likely-transferable (data-driven) findings

Findings tied to the data rather than the continuous loss are candidates for transfer to a classification version of the task: the dominance of temporal distribution shift (so temporal validation is required, not random k-fold); the need for heavy regularization to generalize across drift; the unsafety of library-default early stopping on temporal data; and the value of farthest-point coresets for fair fixed-budget comparison (the coresets are label-agnostic and can be reused directly). Table A.4 grades each by confidence.

### 5.2 Regression-specific (loss-driven) findings

Other findings are tied to the continuous loss and should not be assumed for classification. The R² ≈ 0.62 information ceiling measures continuous-variance explained; a coarser classification target (e.g., gas presence) may be far more separable and is not bounded by it. The dominance of smooth global functions, and the model ranking itself, may invert — class boundaries can be sharp and local, precisely where the tree and boosted methods that lost here may win.

### 5.3 Predictions for a classification pipeline

If the same fairness, temporal-validation, regularization, and coreset principles are applied to a classification version, we would expect: strong achievable accuracy unbounded by the regression ceiling; a re-ordered leaderboard more favourable to tree and boosted methods; persistent temporal-drift fragility concentrated on the same folds; and FPS coresets remaining valuable but possibly needing class stratification. The transferable assets are the methodology and design choices; the non-transferable assets are the specific ceiling and ranking. These are predictions for the partner classification effort, not results of this study.

---

## 6. Conclusion and Future Work

### 6.1 Summary of contributions

Under a fair, temporally-validated, budget-aware protocol, the recommended model for methane and ethylene regression from this 16-sensor array is a regularized cubic spline trained on a farthest-point coreset, reaching pooled R² ≈ 0.62 — the highest in the study. Its architecture, a spline basis with a linear head, is structurally simple, so we expect it to be inexpensive at inference; direct latency and model-size measurements are not yet available (Sections 6.2–6.3). By exhausting two independent levers for improvement, fairer data selection and a richer architecture, and finding neither breaks the ceiling, we establish that R² ≈ 0.62 is an information bound of the current inputs under temporal validation rather than a modeling deficiency.

### 6.2 Limitations

The study is single-seed; multi-seed confidence intervals are not yet attached to the headline numbers. Training R² was not logged, so the overfit/underfit discussion is inferred from fold variance rather than a direct train/validation gap. Hyperparameter configurations were selected on validation folds (non-nested), which can mildly favour the selected configuration. HistGradientBoosting's regression under tuning is attributable to its early-stopping mechanism and should be re-checked with a manual temporal carve before that model is dismissed.

### 6.3 Future work

The next phase should convert the winning model into a deployable artifact: attach multi-seed confidence intervals, profile inference latency and model size, serialize the spline-plus-scaler pipeline, and establish drift monitoring in production (since temporal drift is the dominant failure mode). Because the ceiling is an information bound, the most promising route to higher accuracy is additional information — more sensors or validated temporal-context features — rather than new model architectures. The detailed roadmap is given in the project's closing report (`reports/week4_closing_report.md`).

---

## Appendix A — Supplementary Material

**A.1 Exploratory data analysis.** Figure A.1 (per-sensor distributions, `02_sensor_boxplot.png`); Figure A.2 (baseline vs active sensor response, `02_sensor_response_baseline_vs_active.png`); Figure A.3 (inter-sensor correlation, `02_sensor_sensor_correlation.png`); Figure A.4 (sensor time series, `02_sensor_timeseries.png`).

**A.2 Preprocessing.** Figure A.5 (per-fold scaling, `04_fold1_scaling.png`); Figure A.6 (fair-subset construction, `05_subset_comparison.png`).

**A.3 Coreset detail.** Figure A.7 (construction diagnostics 2×2, `12_coreset_regime_coverage.png`, `12_coreset_sensor_drift.png`, `12_coreset_dispersion.png`, `12_coreset_temporal_density.png`); Figure A.8 (sensitivity heatmap, `13_sensitivity_heatmap.png`); Figure A.9 (coreset per-target, `13_per_target_breakdown.png`); Table A.1 (cross-protocol leakage audit, `13_cross_protocol_audit.parquet`); Table A.2 (full construction diagnostics, `12_coreset_diagnostics_summary.parquet`).

**A.4 Prototype detail.** Figure A.10 (per-fold comparison, `14_per_fold_comparison.png`); Figure A.11 (per-target breakdown, `14_per_target_breakdown.png`).

**A.5 Untuned-stage detail.** Figures A.12–A.16 (per-stage diagnostic panels, `06_baselines.png` … `10_neural_models.png`).

**A.6 Tuning detail.** Table A.3 (search coverage: 2,350 configurations, `11_tuning_search_log.parquet`).

**A.7 Transferability.** Table A.4 (transfer-strength summary, from `reports/week3_task34_transferability_note.md` §6).

---

*Draft v1 ends. Prose follows the frozen outline `reports/paper_master_outline.md`; no section ordering, figure numbering, table numbering, or narrative-spine changes were made. Numeric values cited from saved artifacts. Next drafting iterations: expand 4.2 per-family discussion if venue length permits, render appendix tables in full, and finalize the abstract length to target venue.*
