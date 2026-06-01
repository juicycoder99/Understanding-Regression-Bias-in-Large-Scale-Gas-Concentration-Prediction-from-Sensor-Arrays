# Project

Methane and ethylene regression benchmark using gas sensor-array data.

Current phase: Scientific writing and documentation.

The main experimentation phase is complete. Prefer analysis, interpretation, reporting, and artifact auditing before proposing new experiments.

---

# Project Structure

- Raw data: `data/raw/`
- Processed data: `data/processed/`
- Subsets: `data/subsets/`
- Notebooks: `notebooks/`
- Tables: `results/tables/`
- Figures: `results/figures/`
- Models: `results/models/`
- Memos: `results/memos/`
- Reports: `reports/`

---

# Research Rules

- Preserve temporal order.
- Never shuffle before temporal validation.
- Use fixed seeds when randomness is involved.
- Save every significant result as an artifact.
- All model comparisons must use the same validation protocol and metrics.
- Do not make scientific claims unless supported by saved artifacts.
- Do not invent metrics, rankings, or conclusions.

---

# Writing Rules

- Use saved artifacts as the source of truth.
- Read relevant notebooks, tables, figures, memos, and reports before writing.
- Distinguish clearly between:
  - baseline results
  - tuned results
  - coreset results
  - prototype results
- Explain why results change between stages.
- Prefer synthesis and interpretation over new experimentation.

---

# Workflow Status

Completed:
- Fair benchmark protocol
- Rolling temporal validation
- Hyperparameter tuning
- Coreset construction
- Coreset comparison
- Creative model proposal
- Prototype implementation
- Prototype evaluation

Remaining:
- Task 3.4 (transferability note)
- Task 4.3 (paper structure repair)
- Task 4.4 (closing report)
- Paper writing

---

# Gotchas

- `Comparative Analysis/` is legacy work and reference only.
- Do not reuse old code unless explicitly requested.
- Do not compare results from different dataset versions or validation protocols.
- `time_s` is for ordering, not an automatic model feature.
- Inputs: `s01`–`s16`
- Targets: `methane_ppm`, `ethylene_ppm`

---

# Approval Rules

- Explain the plan before making significant changes.
- Do not delete, rename, or overwrite important files without approval.
- Before proposing new experiments, explain why existing artifacts are insufficient.

---

# Key Files

- Supervisor plan: `docs/Jibril_One_Month_Plan.pdf`
- Main dataset: `data/processed/ethylene_methane.parquet`
- Validation protocol: `results/tables/validation_splits.parquet`