# Paper Master Outline — Final Writing Blueprint

**Date:** 2026-05-29
**Project:** Methane and ethylene regression benchmark from 16-sensor gas-array data.
**Status:** 🔒 **FROZEN (2026-05-29).** This is the authoritative, frozen manuscript structure. All drafting must follow it. Section ordering, figure numbering, table numbering, and the narrative spine are locked and must NOT be changed — the sole exception is a contradiction discovered in the saved artifacts, which must be documented in the Change Log below before any structural edit. No prose in this file.

**Change Log:** *(none — outline frozen as authored 2026-05-29)*
**Sources of truth:** `reports/week4_paper_structure.md` (section/claim blueprint) and `reports/week4_figure_table_catalog.md` (numbered floats, captions, placements). Where the two are combined here, those documents remain the detailed references.
**Scope:** Current-project artifacts only. The legacy `Comparative Analysis/` report is excluded — not cited, repaired, or reused.

---

## 0. How to use this outline

Each section lists, in drafting order:
- **Subsections** with their numbered headings.
- **Claims** (C-numbers carried from `week4_paper_structure.md`) the subsection must support.
- **Floats placed here** (Figure/Table numbers from `week4_figure_table_catalog.md`), with the anchor float in **bold**.
- **Hand-off** — the question this subsection leaves for the next (the narrative spine).

Drafting rule: every main-paper float is introduced in prose before it appears; phase labels in captions must match the surrounding text; all quoted numbers must match the cited source artifact.

---

## Front matter

- **Title** (working): *A Fair, Temporally-Validated Benchmark for Gas-Concentration Regression from Sensor Arrays, with a Diagnostic Information-Ceiling Analysis.*
- **Abstract** — drafted last; must state: fair temporal protocol, spline winner (R² ≈ 0.62), FPS coreset result, prototype rejection, information-bound conclusion.
- **Keywords** — gas sensor array, regression, temporal validation, coreset selection, methane, ethylene, model benchmarking.

---

## Section 1 — Introduction
- **Subsections:** 1 (no sub-split).
- **Claims:** C1.1 (problem: deployment-time gas-concentration prediction), C1.2 (contribution: fair temporally-validated budget-aware benchmark + ceiling diagnostic).
- **Floats placed here:** none.
- **Hand-off:** "what data and protocol make the benchmark fair?" → Section 2/3.

---

## Section 2 — Data and Problem Setup
- **Subsections:** 2.1 Dataset and targets · 2.2 Sensor–target structure · 2.3 Operating regimes and temporal non-stationarity.
- **Claims:** C2.1 (4.18 M rows, 16 sensors, 2 targets), C2.2 (regime shares 32.9/22.8/24.0/20.3), C2.3 (target scale/distribution differ), C2.4 (`time_s` orders, not a feature).
- **Floats placed here:**
  - **Figure 1** (target distributions + temporal profile) — anchor.
  - **Figure 2** (sensor–target correlation + regime map).
- **Hand-off:** "given this temporal structure, how is fairness and leakage controlled?" → Section 3.

---

## Section 3 — Methodology
- **Subsections:** 3.1 Rolling temporal validation · 3.2 Common training budget · 3.3 Per-fold preprocessing · 3.4 Leakage control and early stopping · 3.5 Reproducibility.
- **Claims:** C3.1 (5 temporal folds, train precedes val, no shuffle), C3.2 (common fair-subset budget), C3.3 (fresh per-fold RobustScaler), C3.4 (manual temporal early-stopping carve), C3.5 (SEED=42; every result an artifact).
- **Floats placed here:**
  - **Table 1** (validation protocol windows) — anchor.
  - **Figure 3** (rolling temporal fold layout).
- **Hand-off:** "under this contract, what can models actually achieve?" → Section 4.

---

## Section 4 — Experimental Results and Discussion (main section)

### 4.1 Fair benchmark protocol and baselines
- **Claims:** C4.1.1 (Ridge reference 0.605; methane 0.631, ethylene 0.579), C4.1.2 (mean floor R² < 0).
- **Floats:** **Table 2** (untuned model survey — anchor; contains the baseline rows).
- **Hand-off:** "can anything beat the linear floor?"

### 4.2 Untuned model survey
- **Claims:** C4.2.1 (linear family ≤ Ridge; Huber 0.564), C4.2.2 (poly 0.600 / spline 0.591 approach ceiling; trees/KNN collapse), C4.2.3 (boosted underfit; best 0.374), C4.2.4 (neural underperform; TabNet 0.214).
- **Floats:** **Table 2** (shared anchor with 4.1); detail panels Figures A.12–A.16 (appendix, referenced here once).
- **Hand-off:** "is the collapse fixable by tuning?"

### 4.3 Tuned-model evaluation
- **Claims:** C4.3.1 (spline_tuned wins, 0.619), C4.3.2 (gains = iteration count + regularization + batch/optimizer), C4.3.3 (HGBR regresses 0.325→0.291, early-stopping caution), C4.3.4 (≈0.62 ceiling over 2,350 configs).
- **Floats:** **Figure 4** (tuned R² + ceiling — anchor), **Table 3** (tuned publication comparison); Table A.3 (search-log, appendix reference).
- **Hand-off:** "is the ceiling a data limit or a fairness/sample artifact?"

### 4.4 Coreset construction and fair-budget comparison
- **Claims:** C4.4.1 (per-fold leakage-clean construction, 100 artifacts), C4.4.2 (k-means most representative, density worst, FPS most space-filling), C4.4.3 (FPS wins 10/12; FPS+spline@395k = 0.6221), C4.4.4 (partition models peak small), C4.4.5 (cross-protocol audit benign).
- **Floats:** **Figure 5** (method ranking — anchor), **Figure 6** (budget curves), **Table 4** (sensitivity + winners); Figure A.7 (construction diagnostics), Figure A.8 (sensitivity heatmap), Figure A.9 (per-target), Table A.1 (cross-protocol audit), Table A.2 (full diagnostics) — all appendix, referenced here.
- **Hand-off:** "can a creative architecture break the ceiling?"

### 4.5 Creative prototype and its rejection
- **Claims:** C4.5.1 (spline + boosting residual on FPS coreset, evidence-selected), C4.5.2 (H1–H4 all fail; V1 0.606 < V0 0.622), C4.5.3 (residuals unpredictable), C4.5.4 (ceiling upgraded to information bound).
- **Floats:** **Figure 7** (prototype vs baselines — anchor), **Figure 8** (residual structure), **Table 5** (H1–H4 verdicts); Figure A.10 (per-fold), Figure A.11 (per-target) — appendix, referenced here.
- **Hand-off:** "what is robust across phases, and what does it mean for deployment?"

### 4.6 Synthesis and discussion
- **Claims:** C4.6.1 (stable findings: ceiling, smooth>partition>neural, methane>ethylene, drift dominant, fold-1 ethylene weak), C4.6.2 (changed findings reconciled: spline overtakes Ridge; boosted underfit-not-fail), C4.6.3 (deployment reading: best model is lightest; partition models need less data).
- **Floats:** **Table 6** (cross-phase leaderboard — anchor).
- **Hand-off:** "do these data-driven findings transfer beyond regression?"

---

## Section 5 — Transferability Outlook
- **Subsections:** 5.1 Likely-transferable (data-driven) findings · 5.2 Regression-specific (loss-driven) findings · 5.3 Predictions for a classification pipeline.
- **Claims:** C5.1 (drift, regularization, early-stopping pitfall, FPS coresets transfer), C5.2 (R² ceiling, smooth dominance, model ranking may not transfer), C5.3 (enumerated testable predictions).
- **Floats placed here:** **Table A.4** (transfer-strength summary — appendix, referenced as the anchor for this section).
- **Hand-off:** "what is the bottom line and the next-month roadmap?"

---

## Section 6 — Conclusion and Future Work
- **Subsections:** 6.1 Summary of contributions · 6.2 Limitations · 6.3 Future work.
- **Claims:** C6.1 (headline: FPS-selected spline_tuned ≈ 0.62, ceiling is an information bound), C6.2 (limitations: single-seed, no train/val-gap logging, val-based config selection, HGBR caveat), C6.3 (roadmap — references Task 4.4 closing report).
- **Floats placed here:** none (optional compact summary table may reuse Table 6).
- **Hand-off:** end of manuscript.

---

## Appendix A — Supplementary Material

| location | float | source file | from catalog |
|---|---|---|---|
| A.1 EDA | Figure A.1 | `02_sensor_boxplot.png` | sensor distributions |
| A.1 EDA | Figure A.2 | `02_sensor_response_baseline_vs_active.png` | baseline vs active |
| A.1 EDA | Figure A.3 | `02_sensor_sensor_correlation.png` | inter-sensor correlation |
| A.1 EDA | Figure A.4 | `02_sensor_timeseries.png` | sensor time series |
| A.2 Preprocessing | Figure A.5 | `04_fold1_scaling.png` | per-fold scaling |
| A.2 Preprocessing | Figure A.6 | `05_subset_comparison.png` | fair-subset construction |
| A.3 Coreset detail | Figure A.7 | `12_coreset_{regime_coverage,sensor_drift,dispersion,temporal_density}.png` | construction diagnostics (2×2) |
| A.3 Coreset detail | Figure A.8 | `13_sensitivity_heatmap.png` | sensitivity heatmap |
| A.3 Coreset detail | Figure A.9 | `13_per_target_breakdown.png` | coreset per-target |
| A.3 Coreset detail | Table A.1 | `13_cross_protocol_audit.parquet` | leakage audit |
| A.3 Coreset detail | Table A.2 | `12_coreset_diagnostics_summary.parquet` | full diagnostics |
| A.4 Prototype detail | Figure A.10 | `14_per_fold_comparison.png` | prototype per-fold |
| A.4 Prototype detail | Figure A.11 | `14_per_target_breakdown.png` | prototype per-target |
| A.5 Untuned detail | Figures A.12–A.16 | `06`–`10` `*.png` | per-stage diagnostic panels |
| A.6 Tuning detail | Table A.3 | `11_tuning_search_log.parquet` | search coverage |
| A.7 Transfer | Table A.4 | `week3_task34_transferability_note.md` §6 | transfer-strength summary |

---

## Float location index (every numbered float → final home)

### Main figures
| float | section | role |
|---|---|---|
| Figure 1 | 2.1 | anchor |
| Figure 2 | 2.2 | support |
| Figure 3 | 3.1 | support |
| Figure 4 | 4.3 | anchor (ceiling: observed) |
| Figure 5 | 4.4 | anchor |
| Figure 6 | 4.4 | support |
| Figure 7 | 4.5 | anchor |
| Figure 8 | 4.5 | support (ceiling: explained) |

### Main tables
| float | section | role |
|---|---|---|
| Table 1 | 3.1 | anchor |
| Table 2 | 4.1 / 4.2 | anchor (shared) |
| Table 3 | 4.3 | support |
| Table 4 | 4.4 | support |
| Table 5 | 4.5 | support |
| Table 6 | 4.6 | anchor (synthesis) |

### Appendix floats
| float | appendix loc | section referenced from |
|---|---|---|
| Figure A.1–A.4 | A.1 | 2 |
| Figure A.5–A.6 | A.2 | 3 |
| Figure A.7–A.9 | A.3 | 4.4 |
| Figure A.10–A.11 | A.4 | 4.5 |
| Figure A.12–A.16 | A.5 | 4.2 |
| Table A.1–A.2 | A.3 | 4.4 |
| Table A.3 | A.6 | 4.3 |
| Table A.4 | A.7 | 5 |

---

## Narrative spine (the through-line to preserve while drafting)

```
2 Data ─► 3 Methodology
        │
        ▼
4.1 Baselines: linear floor (Ridge 0.605) ............ Table 2
        │  can anything beat linear?
        ▼
4.2 Untuned survey: smooth approaches ceiling, others collapse
        │  fixable by tuning?
        ▼
4.3 Tuning: spline wins 0.619; ceiling appears ....... Figure 4 + Table 3   [ceiling: OBSERVED]
        │  data limit or sample artifact?
        ▼
4.4 Coreset: FPS helps (0.6221), ceiling holds ....... Figures 5–6 + Table 4 [ceiling: SURVIVES selection]
        │  can a creative architecture break it?
        ▼
4.5 Prototype rejected: residuals are noise .......... Figures 7–8 + Table 5 [ceiling: EXPLAINED as info bound]
        │  what is robust; what for deployment?
        ▼
4.6 Synthesis ........................................ Table 6
        │  does it transfer?
        ▼
5 Transferability ─► 6 Conclusion
```

**Ceiling thread:** Figure 4 (observed) → Figures 5–6 (survives better data selection) → Figure 8 + Table 5 (explained as information bound). Each later mention must reference the earlier one. This escalation is the manuscript's central argument.

---

## Drafting-pass checklist (per section, applied at writing time)

- [ ] Every claim (C-number) addressed and backed by its cited artifact.
- [ ] Anchor float introduced in prose before it appears.
- [ ] Phase label in each caption matches surrounding prose.
- [ ] All quoted numbers match the source artifact in `week4_figure_table_catalog.md`.
- [ ] Hand-off sentence present, posing the next subsection's question.
- [ ] No legacy-project content referenced.

---

**Sign-off.** This master outline consolidates the section blueprint and the float catalog into one writing-ready structure: every section and subsection mapped to its claims and floats, every numbered figure/table assigned a final main-paper or appendix home, and the narrative spine and ceiling-thread made explicit. Prose drafting is the next pass. Task 4.4 (closing report) remains a separate deliverable.
