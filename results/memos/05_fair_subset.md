# 05 - Fair Subset (1M-Row Common Training Budget)

**Date:** 2026-05-22
**Dataset:** ../data/processed/ethylene_methane.parquet
**Splits artifact:** ../results/tables/validation_splits.parquet
**Subset artifact:** ../data/subsets/fair_subset_indices.parquet

---

## Approved decisions
- Budget: **1,000,000 rows** global pool (~23.9% of dataset).
  - Each fold trains on `train_range ∩ subset`, which is ~394,987 rows per fold (≈ 9.5% of dataset, ≈ 39% of the pool). The 1M figure is the shared pool, not the per-fold training size.
- Temporal order preserved (indices sorted by `time_s`).
- 4-way `(methane>0, ethylene>0)` regime balance preserved.
- Applies to **training windows only**; validation slices remain full.
- Single global subset artifact, intersected with each fold train range at model-training time.
- 06 baselines refit per-fold scalers on `train_range ∩ subset` (strict fairness).

## Chosen strategy: **stratified**

Stratified sampling preserves the 4-way regime balance by construction. Chronological downsampling is reported alongside as a sanity baseline; at this dataset scale and bin interleaving the two strategies produce indistinguishable proportions, but stratified retains the guarantee if data composition shifts.

## Strategy comparison (regime balance)

| subset | n_rows | bin_00_pct | bin_01_pct | bin_10_pct | bin_11_pct |
| --- | --- | --- | --- | --- | --- |
| FULL | 4178504 | 0.3289 | 0.2284 | 0.2396 | 0.2031 |
| chronological | 1000000 | 0.3289 | 0.2284 | 0.2396 | 0.2031 |
| stratified | 1000000 | 0.3289 | 0.2284 | 0.2396 | 0.2031 |

## Per-fold train_window ∩ subset

| fold | train_window_rows | subset_in_train | subset_pct_of_train |
| --- | --- | --- | --- |
| 1 | 1664421 | 398670 | 0.2395 |
| 2 | 1634650 | 391532 | 0.2395 |
| 3 | 1641557 | 392782 | 0.2393 |
| 4 | 1641541 | 392275 | 0.239 |
| 5 | 1671616 | 399678 | 0.2391 |

## Usage (downstream notebooks)

```python
subset_idx = pd.read_parquet("data/subsets/fair_subset_indices.parquet")["row_idx"].to_numpy()
for fold_id in splits_df["fold"].unique():
    tr = splits_df[(splits_df["fold"] == fold_id) & (splits_df["split"] == "train")].iloc[0]
    va = splits_df[(splits_df["fold"] == fold_id) & (splits_df["split"] == "val")].iloc[0]
    train_idx = subset_idx[(subset_idx >= tr["start_idx"]) & (subset_idx < tr["end_idx"])]
    X_train   = df[SENSORS].iloc[train_idx].to_numpy()
    X_val     = df[SENSORS].iloc[va["start_idx"]:va["end_idx"]].to_numpy()  # full val, NOT subset
```

## Next step

Notebook `06_baselines.ipynb` (pending approval): refit per-fold scalers on `train_range ∩ subset`, train baseline models on the fair subset, evaluate on full validation slices.