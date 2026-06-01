# 03 — Validation Split Protocol

**Date:** 2026-05-18
**Dataset:** ../data/processed/ethylene_methane.parquet
**Artifact:** ../results/tables/validation_splits.parquet

---

## Protocol
- Window kind: **rolling**
- Number of folds: **5**
- Train fraction (per fold): 0.4
- Validation fraction (per fold): 0.1
- Split boundaries aligned to unique timestamps (tie-safe)
- No shuffling, no random sampling, temporal order preserved

## Verification
- All folds have non-overlapping train/val row ranges: True
- All folds have non-overlapping train/val time ranges: True
- Both targets have non-zero rows in train and val for every fold: True

## Fold Summary

| fold | train_rows | val_rows | tr_methane_nonzero | tr_ethylene_nonzero | va_methane_nonzero | va_ethylene_nonzero |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | 1664421 | 407367 | 809569 | 686487 | 226902 | 177001 |
| 2 | 1634650 | 419449 | 795778 | 671520 | 218023 | 273968 |
| 3 | 1641557 | 407362 | 723460 | 735888 | 194761 | 77500 |
| 4 | 1641541 | 431972 | 813971 | 646077 | 87018 | 219185 |
| 5 | 1671616 | 435098 | 684382 | 784770 | 128899 | 154702 |

## Usage

```python
splits_df = pd.read_parquet("results/tables/validation_splits.parquet")
for fold_id in splits_df.fold.unique():
    tr = splits_df.query("fold == @fold_id and split == \"train\"").iloc[0]
    va = splits_df.query("fold == @fold_id and split == \"val\"").iloc[0]
    train = df.iloc[tr.start_idx:tr.end_idx]
    val   = df.iloc[va.start_idx:va.end_idx]
    # ... fit on train, evaluate on val ...
```

## Next Step

Notebook `04_preprocessing.ipynb`: define scaling/transform pipeline that fits on **training fold only** and applies to validation fold. No leakage.