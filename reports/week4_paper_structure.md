# Task 4.3 — Paper Structure Blueprint

**Date:** 2026-05-29
**Project:** Methane and ethylene regression benchmark from 16-sensor gas-array data.
**Status:** Annotated outline only. No paper prose drafted. Closes Week 4, Task 4.3 (reframed for the current project).
**Scope constraint:** The current project artifacts are the sole source of truth. The legacy `Comparative Analysis/` report is out of scope — not cited, repaired, or reused. The One-Month Plan's references to repairing Section 4, removing Sections 5–6, and reconciling Table 1 vs Table 12 are applied here only as a *methodological lesson* (one coherent experimental section; explicit baseline / tuned / coreset / prototype phase labels), not as instructions about the legacy text.

---

## 1. How to read this blueprint

This is a **construction plan for the paper**, not the paper. For every section it records:
- **Claim(s):** the scientific statement(s) the section will make.
- **Source artifact(s):** the exact saved file(s) that license each claim (memo, table, figure, report). Nothing is claimed without a backing artifact.
- **Figures/tables to embed:** which saved figures and which numbers belong in the section.
- **Phase label:** baseline / tuned / coreset / prototype — every result is tagged so a reader never confuses phases.

The actual prose is deferred to a later writing pass. Numbers shown below are anchors copied from saved artifacts so the writer does not have to re-derive them; they are not new results.

---

## 2. Top-level section map

| § | title | role | primary phase |
|---|---|---|---|
| 1 | Introduction | problem, contribution, Pergamon framing | — |
| 2 | Data and Problem Setup | dataset, targets, sensors, temporal structure | — |
| 3 | Methodology | protocol, validation, fairness, leakage control | — |
| 4 | Experimental Results and Discussion | the single main results section (all phases as subsections) | all |
| 5 | Transferability Outlook | regression → classification bridge | — |
| 6 | Conclusion and Future Work | findings, limitations, roadmap | — |

**Methodological lesson applied:** all experimental results live inside Section 4 as coherent subsections (4.1–4.6). There is no separate top-level "more results" section. This mirrors the lesson from the One-Month Plan without referencing the legacy report.

---

## 3. Section-by-section blueprint

### Section 1 — Introduction
- **Claims:**
  - C1.1 Predicting gas concentrations from a 16-sensor array under realistic (future-time) deployment is the target problem.
  - C1.2 The contribution is a *fair, temporally-validated, budget-aware* benchmark plus a diagnostic of why models degrade and where the predictive ceiling lies.
- **Source artifacts:** `CLAUDE.md` (problem framing); `reports/week1_fairness_memo.md` (contribution statement).
- **Figures/tables:** none.
- **Phase label:** —
- **Narrative role:** states the question and the three pillars (fair comparison, temporal evaluation, diagnostic-to-prototype) that structure the paper.

### Section 2 — Data and Problem Setup
- **Claims:**
  - C2.1 Dataset is 4,178,504 rows × {time_s, s01–s16, methane_ppm, ethylene_ppm}.
  - C2.2 Four operating regimes exist by (methane>0, ethylene>0): 32.9% / 22.8% / 24.0% / 20.3%.
  - C2.3 Targets differ in scale and distribution (methane wider range).
  - C2.4 `time_s` orders the data and must not be used as a feature.
- **Source artifacts:** `results/memos/01_inspection_summary.md`, `results/memos/02_eda_summary.md`, `results/memos/05_fair_subset.md` (regime proportions).
- **Figures/tables:** embed `02_target_distributions.png`, `02_target_timeseries.png`, `02_sensor_target_correlation.png`, `02_regime_map.png`. Table: regime proportions (from 05 memo).
- **Phase label:** —
- **Narrative role:** establishes what the data is and the temporal/regime structure that later explains drift and the ethylene difficulty.

### Section 3 — Methodology
- **Claims:**
  - C3.1 Evaluation uses 5 rolling temporal folds; train always precedes val; no shuffling.
  - C3.2 A common training budget (fair subset) is enforced so all models see the same data volume.
  - C3.3 A fresh RobustScaler is fit per fold on training data only.
  - C3.4 Leakage is controlled at every stage; early stopping uses a manual temporal carve.
  - C3.5 SEED=42 throughout; every result is a saved artifact.
- **Source artifacts:** `reports/week1_fairness_memo.md` (the canonical protocol statement), `results/memos/03_validation_split.md`, `results/memos/04_preprocessing.md`, `results/memos/05_fair_subset.md`, `results/tables/validation_splits.parquet`.
- **Figures/tables:** embed `03_fold_layout.png`, `04_fold1_scaling.png`, `05_subset_comparison.png`. Table: per-fold train/val window sizes (from validation_splits).
- **Phase label:** —
- **Narrative role:** the methodological contract every result in Section 4 inherits. Written once, referenced thereafter.

### Section 4 — Experimental Results and Discussion (main section)

#### 4.1 Fair benchmark protocol and baselines
- **Claims:**
  - C4.1.1 Ridge sets the linear reference at pooled R² = 0.605 (methane 0.631, ethylene 0.579).
  - C4.1.2 The mean baseline confirms the floor (R² < 0).
- **Source artifacts:** `results/memos/06_baselines.md`, `results/tables/06_baseline_metrics_long.parquet` / `_summary`.
- **Figures/tables:** `06_baselines.png`; baseline ranking table.
- **Phase label:** **baseline.**
- **Narrative role:** anchors the scale; every later number is read against Ridge 0.605.

#### 4.2 Untuned model survey (linear, nonlinear, boosted, neural)
- **Claims:**
  - C4.2.1 Linear/statistical family does not beat Ridge; Huber best at 0.564 (Stage 07).
  - C4.2.2 Smooth nonlinear (poly 0.600, spline 0.591) approaches the ceiling; trees/KNN collapse under drift (Stage 08).
  - C4.2.3 Boosted models underfit at 100 iterations (best GBR 0.374; XGB worst 0.066) (Stage 09).
  - C4.2.4 Neural models underperform smooth methods (TabNet 0.214, MLP 0.069) (Stage 10).
- **Source artifacts:** memos `07`–`10`; tables `07`–`10` long/summary.
- **Figures/tables:** `07_linear_models.png`, `08_nonlinear_models.png`, `09_boosted_models.png`, `10_neural_models.png`; a consolidated untuned-ranking table.
- **Phase label:** **baseline (untuned).**
- **Narrative role:** establishes the inductive-bias landscape and motivates tuning; flags temporal drift as the recurring failure.

#### 4.3 Tuned-model evaluation
- **Claims:**
  - C4.3.1 spline_tuned is the tuned winner at pooled R² = 0.619 (methane 0.661, ethylene 0.578).
  - C4.3.2 Tuning's gains trace to three mechanisms: iteration count, regularization strength, batch/optimizer regime.
  - C4.3.3 HistGradientBoosting regressed under tuning (0.325 → 0.291) due to non-temporal built-in early stopping — a methodological caution.
  - C4.3.4 The R² ≈ 0.62 ceiling persists across 2,350 configurations.
- **Source artifacts:** `results/memos/11_tuning_optimization.md`, `results/tables/11_tuning_metrics_long.parquet` / `_summary` / `11_tuning_search_log.parquet`; `reports/week1_table_audit.md` (baseline→tuned reconciliation, stable vs changed findings).
- **Figures/tables:** `11_tuning_optimization.png`; the publication comparison table (10 tuned models, per-target R², MAE, RMSE) from the table audit.
- **Phase label:** **tuned.**
- **Narrative role:** the headline benchmark result; introduces the ceiling as an empirical observation (later upgraded to an information bound in 4.5).

#### 4.4 Coreset construction and fair-budget comparison
- **Claims:**
  - C4.4.1 Coresets are constructed per-fold from train ranges only (leakage-clean Option C); 100 artifacts validated.
  - C4.4.2 Construction diagnostics: k-means most representative (sensor-mean dev 0.001); density worst (sensor-std inflation up to 4.62×); FPS most space-filling.
  - C4.4.3 Under model evaluation, FPS wins 10/12 cells; best overall FPS+spline@395k = 0.6221.
  - C4.4.4 Partition models peak below full budget (CatBoost@25k = 0.5275, ExtraTrees@60k = 0.5834); smooth models rise monotonically.
  - C4.4.5 Cross-protocol audit: per-fold leakage-clean stratified vs Stage-05 global stratified agree within fold-std → prior leakage was numerically benign.
- **Source artifacts:** `results/memos/12_coreset_strategies.md` (incl. the Construction Pool Design Decision section), `results/memos/13_coreset_comparison.md`; tables `12_coreset_diagnostics_*`, `13_coreset_comparison_*`, `13_coreset_sensitivity`, `13_budget_curves`, `13_cross_protocol_audit`.
- **Figures/tables:** `12_coreset_regime_coverage.png`, `12_coreset_sensor_drift.png`, `12_coreset_dispersion.png`, `12_coreset_temporal_density.png`, `13_method_ranking.png`, `13_budget_curves.png`, `13_sensitivity_heatmap.png`, `13_per_target_breakdown.png`; sensitivity and cross-protocol-audit tables.
- **Phase label:** **coreset.**
- **Narrative role:** answers the fairness/budget question and sets up the prototype (which coreset to build on, which model is coreset-sensitive).

#### 4.5 Creative prototype and its rejection
- **Claims:**
  - C4.5.1 The proposed direction was spline + boosting residual on an FPS coreset, selected from four archetypes on evidence.
  - C4.5.2 The prototype failed all four pre-registered hypotheses (H1–H4): primary hybrid V1 = 0.606 < spline-only V0 = 0.622; ExtraTrees-residual V3 = 0.624 (within noise).
  - C4.5.3 The residuals are unpredictable: a model that scores 0.51 on raw targets extracts no signal from spline residuals.
  - C4.5.4 This upgrades the R² ≈ 0.62 ceiling from empirical observation to a demonstrated information bound of the 16-sensor inputs under temporal validation.
- **Source artifacts:** `reports/week3_task33_creative_direction.md` (selection + pre-registered hypotheses), `results/memos/14_prototype_v1.md`; tables `14_prototype_metrics_*`, `14_prototype_ablation`, `14_per_fold_breakdown`.
- **Figures/tables:** `14_prototype_vs_baselines.png`, `14_residual_structure.png`, `14_per_fold_comparison.png`, `14_per_target_breakdown.png`; H1–H4 verdict table; ablation table.
- **Phase label:** **prototype.**
- **Narrative role:** the scientific climax — a rigorously rejected hypothesis that yields the paper's strongest conclusion (the information bound). A negative result presented as a positive contribution.

#### 4.6 Synthesis and discussion
- **Claims:**
  - C4.6.1 Stable findings across all phases: ceiling ≈ 0.62; smooth > partition > neural; methane easier than ethylene; temporal drift is the dominant failure; fold-1 ethylene is a structural weak point.
  - C4.6.2 Changed findings after tuning (e.g., spline overtakes Ridge; boosted underfit-not-fail) are reconciled.
  - C4.6.3 Deployment reading: the best model is also the lightest (spline), and partition models need less data — relevant to Pergamon.
- **Source artifacts:** `reports/week1_table_audit.md` (stable vs changed findings), all Section-4 memos.
- **Figures/tables:** a single cross-phase leaderboard table (baseline → tuned → coreset → prototype best per model).
- **Phase label:** all (explicitly labeled per row).
- **Narrative role:** ties the four phases into one story and states what is robust vs phase-dependent.

### Section 5 — Transferability Outlook
- **Claims:**
  - C5.1 Data-driven findings (temporal drift, regularization need, early-stopping pitfall, FPS coresets) are likely to transfer to a classification pipeline.
  - C5.2 Loss-driven findings (the R² ceiling, smooth-method dominance, the model ranking) are regression-specific and may not transfer.
  - C5.3 Concrete, testable predictions for a classification version are enumerated.
- **Source artifacts:** `reports/week3_task34_transferability_note.md`.
- **Figures/tables:** transfer-strength summary table (from the note).
- **Phase label:** —
- **Narrative role:** positions the work within the broader Pergamon project (regression + classification convergence) without running classification experiments.

### Section 6 — Conclusion and Future Work
- **Claims:**
  - C6.1 Headline: spline_tuned on an FPS coreset, R² ≈ 0.62, is the recommended deployable model; the ceiling is an information bound.
  - C6.2 Limitations: single-seed, no train/val-gap logging, val-based config selection, HGBR early-stopping caveat.
  - C6.3 Future work / next-month roadmap.
- **Source artifacts:** the Task 4.4 closing report (to be written) supplies C6.1–C6.3; this section will reference it. Limitations are sourced from `reports/week1_fairness_memo.md` §9.
- **Figures/tables:** none (or a compact summary table).
- **Phase label:** —
- **Narrative role:** closes the loop; defers the detailed roadmap to Task 4.4.

---

## 4. Narrative flow between stages (the spine)

The paper tells one continuous story; each subsection hands off to the next:

```
2 Data ──► 3 Methodology (protocol contract)
        │
        ▼
4.1 Baselines establish the linear floor (Ridge 0.605)
        │   "can anything beat linear?"
        ▼
4.2 Untuned survey maps the inductive-bias landscape; smooth approaches the ceiling, others collapse under drift
        │   "is the collapse fixable by tuning?"
        ▼
4.3 Tuning: spline wins (0.619); boosted underfit-not-fail; ceiling ≈ 0.62 holds
        │   "is the ceiling a data limit or a fairness/sample artifact?"
        ▼
4.4 Coreset study: fairer budgets (FPS) help slightly (0.6221) but do not break the ceiling; partition models peak small
        │   "can a creative architecture break it?"
        ▼
4.5 Prototype: spline+residual rejected; residuals are noise → ceiling is an INFORMATION BOUND
        │   "what is robust, and what does it mean for deployment?"
        ▼
4.6 Synthesis ──► 5 Transferability ──► 6 Conclusion
```

**Through-line:** each stage poses the question the next stage answers. The ceiling is introduced as an observation (4.3), pressure-tested by better data selection (4.4), and finally explained as an information bound (4.5). This escalation is the paper's argument.

---

## 5. Figure and table budget (current artifacts only)

**Figures available and placed:** 23 of the 29 saved figures are assigned to sections above. The 6 not placed (`02_sensor_boxplot`, `02_sensor_response_baseline_vs_active`, `02_sensor_sensor_correlation`, `02_sensor_timeseries`, `02_target_timeseries` partial, `04_fold1_scaling` optional) are held as appendix/supplementary candidates.

**Core tables to render in the paper (from saved parquets):**
1. Per-fold train/val sizes — `validation_splits.parquet`.
2. Untuned ranking — `06`–`10` summaries.
3. Tuned publication table — `11_tuning_metrics_summary.parquet` + `week1_table_audit.md`.
4. Coreset method ranking + sensitivity — `13_coreset_comparison_summary`, `13_coreset_sensitivity`.
5. Cross-protocol audit — `13_cross_protocol_audit`.
6. Prototype H1–H4 verdicts + ablation — `14_prototype_*`.
7. Cross-phase leaderboard (4.6) — assembled from `06`–`14` long tables.
8. Transfer-strength summary — `week3_task34_transferability_note.md`.

**Phase labeling rule (applied to every table caption):** each results table caption must state its phase — baseline / tuned / coreset / prototype — and never mix phases in one table without an explicit phase column.

---

## 6. Scope and exclusions (explicit)

- **Legacy report excluded.** No content, table, figure, or claim from `Comparative Analysis/` appears in this structure. There is no "Table 1 vs Table 12" reconciliation in the current paper; the only reconciliation present is the current project's **baseline (06–10) vs tuned (11)** audit, which is a within-project, same-protocol comparison.
- **No new experiments.** Every claim maps to an existing saved artifact listed above.
- **No prose drafted.** This is the blueprint; writing is a later pass.
- **Task 4.4 kept separate.** The closing report (learnings + next-month roadmap) is a distinct deliverable; Section 6 will reference it but this blueprint does not absorb it.

---

## 7. Open decisions for the writing pass (not blocking)

1. Whether 4.2 (untuned survey) is one subsection or split per family — current plan keeps it consolidated to avoid four thin subsections.
2. Whether the cross-phase leaderboard (4.6) is a figure, a table, or both.
3. Appendix scope for the 6 unplaced EDA figures.
4. Target venue/length, which sets how much of 4.2 and 4.4 is compressed into supplementary material.

These are writing-time choices and do not affect the blueprint's validity.

---

**Sign-off.** This blueprint maps every paper section and subsection to current-project artifacts as the sole source of truth, assigns figures/tables, labels result phases, and defines the stage-to-stage narrative spine — without drafting prose and without referencing the legacy report. Ready for review before the writing pass. Task 4.4 (closing report) remains the separate, final deliverable.
