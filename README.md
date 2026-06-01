# Fair, Temporally-Validated Benchmark for Gas-Concentration Regression

Benchmark study predicting **methane (`methane_ppm`)** and **ethylene (`ethylene_ppm`)** concentration from a 16-channel gas-sensor array (`s01`–`s16`), under a deliberately fair, leakage-controlled, temporally-validated protocol.

The experimentation phase is complete. The project is now in the scientific writing and documentation phase.

---

## Headline result

Under a fair rolling-temporal protocol, the strongest model is a **regularized cubic spline trained on a farthest-point (FPS) coreset**, reaching **pooled R² = 0.6221** (methane ≈ 0.66, ethylene ≈ 0.58). No model across 2,350 tuned configurations and 60 coreset variants exceeds **R² ≈ 0.62**. A two-stage spline-plus-boosting prototype was tested and **rejected**: the spline residuals carry no learnable structure, establishing the ceiling as a **fundamental information bound of the 16 sensor inputs under temporal validation**, not a modeling deficiency.

---

## Methodology (the fairness contract)

- **Rolling temporal validation:** 5 folds; each training window strictly precedes its validation window; no shuffling.
- **Common training budget:** all models trained on the same data volume (~395k rows/fold).
- **Per-fold preprocessing:** a fresh `RobustScaler` fit on training data only.
- **Leakage control:** validation windows never used for fitting, tuning, scaling, or early stopping; early stopping uses a manual temporally-latest carve.
- **Reproducibility:** fixed seed (42); every result saved as an artifact.

Full protocol: [`reports/week1_fairness_memo.md`](reports/week1_fairness_memo.md).

---

## Repository structure

```
notebooks/        Analysis notebooks (Stages 01–14) — the source code of the study
results/
  memos/          Per-stage technical memos (committed)
  tables/         Per-stage metric tables, parquet (committed)
  figures/        All figures, png (committed)
  models/         Fitted per-fold model artifacts (NOT committed — see below)
reports/          Cross-cutting reports, manuscript blueprint, draft, closing report (committed)
docs/             Supervisor one-month plan (committed)
data/
  raw/            Raw dataset (NOT committed)
  processed/      Cleaned dataset (NOT committed)
  subsets/        Fair subset + 100 per-fold coresets (NOT committed)
CLAUDE.md         Project rules and working conventions
README.md         This file
```

### What is NOT in the repository (and why)

Large or regenerable artifacts are excluded via `.gitignore` to keep the repo lightweight (~6 MB committed vs ~6.5 GB on disk):

| excluded | size | reason |
|---|---|---|
| `data/raw/`, `data/processed/` | ~740 MB | source data; obtain separately |
| `data/subsets/` | ~160 MB | derived coresets; regenerable from notebooks 05, 12 |
| `results/models/` | ~2.2 GB | fitted models; regenerable from notebooks 06–14 |
| `Comparative Analysis/` | ~3.4 GB | legacy project, reference only — not part of this study |

Excluded directories are preserved as empty folders via `.gitkeep` so the layout is visible after cloning.

---

## Reproducing the artifacts

1. Place the raw dataset at `data/processed/ethylene_methane.parquet` (the notebooks read the processed parquet).
2. Run the notebooks in order:
   - `03` builds the temporal validation splits → `results/tables/validation_splits.parquet`.
   - `05` builds the fair subset; `12` builds the per-fold coresets.
   - `06`–`10` run the untuned benchmark; `11` runs tuning; `13` runs the coreset comparison; `14` runs the prototype.
3. Each notebook writes its tables, figures, fitted models, and a memo. The committed tables/figures/memos let you read every result without re-running.

Environment: Python 3.13; scikit-learn, xgboost, lightgbm, catboost, pytorch-tabnet (CPU). Fixed seed 42 throughout.

---

## Stage map

| stage | notebook | output |
|---|---|---|
| 01–05 | setup | inspection, EDA, temporal splits, preprocessing, fair subset |
| 06–10 | untuned benchmark | baselines, linear, nonlinear, boosted, neural |
| 11 | tuning | 2,350 configs; spline winner (0.619) |
| 12 | coreset construction | 100 leakage-clean per-fold coresets |
| 13 | coreset comparison | FPS wins 10/12; best 0.6221 |
| 14 | prototype | spline+residual; rejected (information bound) |

---

## Key reports

- [`reports/week1_fairness_memo.md`](reports/week1_fairness_memo.md) — benchmark protocol and leakage control.
- [`reports/week1_table_audit.md`](reports/week1_table_audit.md) — baseline vs tuned reconciliation.
- [`reports/week3_task34_transferability_note.md`](reports/week3_task34_transferability_note.md) — regression → classification transfer.
- [`reports/paper_master_outline.md`](reports/paper_master_outline.md) — **frozen** manuscript structure.
- [`reports/manuscript_draft.md`](reports/manuscript_draft.md) — current manuscript draft.
- [`reports/week4_closing_report.md`](reports/week4_closing_report.md) — scientific learnings + next-month roadmap.

---

## Research rules (summary)

Preserve temporal order; never shuffle before temporal validation; fixed seeds; same validation protocol and metrics for all comparisons; no scientific claim without a saved artifact; no invented metrics. Full rules in [`CLAUDE.md`](CLAUDE.md).
