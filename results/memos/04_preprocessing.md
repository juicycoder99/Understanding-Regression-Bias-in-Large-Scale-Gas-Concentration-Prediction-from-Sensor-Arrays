# 04 - Preprocessing (Per-Fold Scaling)

**Date:** 2026-05-19
**Dataset:** ../data/processed/ethylene_methane.parquet
**Splits artifact:** ../results/tables/validation_splits.parquet
**Scaler artifacts:** ../results/models/preprocessors/fold_{1..5}.joblib

---

## Decisions (all user-approved)
- Input scaler: `RobustScaler` (median / IQR) applied to `s01..s16`.
- Targets `methane_ppm`, `ethylene_ppm`: no transformation.
- One fitted scaler persisted per fold; preprocessed arrays NOT cached.
- `time_s` excluded from features.
- No feature engineering in this notebook.

## Anti-leakage
- Each scaler fit only on the train slice of its fold.
- Verification reloaded every scaler, transformed train and val slices, asserted finite values.
- Pairwise scaler-center differences across folds are non-zero (folds genuinely distinct).

## Per-Fold Scaler Summary

| fold | n_train_rows | n_val_rows | center_mean | center_std | scale_mean | scale_std |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | 1664421 | 407367 | 2446.6325 | 497.2733 | 1400.5925 | 717.8931 |
| 2 | 1634650 | 419449 | 2486.8991 | 522.2009 | 1574.4425 | 849.7028 |
| 3 | 1641557 | 407362 | 2351.7931 | 716.6388 | 1594.8762 | 836.5587 |
| 4 | 1641541 | 431972 | 2512.59 | 519.6675 | 1602.3794 | 858.7909 |
| 5 | 1671616 | 435098 | 2286.6725 | 755.1972 | 1614.0881 | 819.8894 |

## Usage

```python
import joblib
art = joblib.load("results/models/preprocessors/fold_1.joblib")
scaler = art["scaler"]
X_train  = df[art["feature_cols"]].iloc[art["train_start_idx"]:art["train_end_idx"]].to_numpy()
X_val    = df[art["feature_cols"]].iloc[art["val_start_idx"]:art["val_end_idx"]].to_numpy()
Xt_train = scaler.transform(X_train)
Xt_val   = scaler.transform(X_val)
# Targets used in raw ppm:
y_train  = df[art["target_cols"]].iloc[art["train_start_idx"]:art["train_end_idx"]].to_numpy()
y_val    = df[art["target_cols"]].iloc[art["val_start_idx"]:art["val_end_idx"]].to_numpy()
```

## Next Step

Notebook `05_baselines.ipynb` (pending approval): train baseline models per fold using these scalers, evaluate with the same metrics across all folds.