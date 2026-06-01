# 01 — Data Inspection Summary

**Date:** 2026-05-18
**Dataset:** data/processed/ethylene_methane.parquet

---

## Shape

- Rows: 4,178,504
- Columns: 19 (`time_s`, `s01`–`s16`, `methane_ppm`, `ethylene_ppm`)

---

## Column Check

- Missing sensors: None ✓
- Missing targets: None ✓
- `time_s` present: True ✓
- Unexpected columns: None ✓

---

## Data Quality

- Null values: None ✓
- Duplicate rows: 0 ✓
- Constant sensors: None ✓
- Near-constant sensors: None ✓

---

## Temporal Order

- `time_s` monotonically increasing: True ✓
- `time_s` tied values: 104,950 (multi-sensor readings at the same timestamp — expected, not an error)
- `time_s` range: 0.00 → 41,790.19

---

## Target Summary

| Target | Min | Max | Mean | % Zero Rows |
|--------|-----|-----|------|-------------|
| `methane_ppm` | 0.00 | 296.67 | 58.09 | 55.7% |
| `ethylene_ppm` | 0.00 | 20.00 | 4.37 | 56.9% |

---

## Key Observations

- Dataset is large (4.18M rows), fully temporally ordered, no missing data — suitable for rolling temporal validation.
- Over half the rows have zero target values, reflecting baseline (no-gas) periods in the experiment. This is a structural property of the data, not noise.
- Tied `time_s` values (104,950) indicate multiple sensor channels logged at the same instant — a known characteristic of the sensor array, not a data error.
- No sensor dropout or constant channels detected across all 16 sensors.
- `methane_ppm` reaches up to 296.67 ppm; `ethylene_ppm` up to 20.00 ppm — very different scales.

---

## Constraints for Next Steps

- **Target balance:** Any reduced subsets or train/test splits must preserve the zero/nonzero ratio for both targets. Do not create splits that accidentally concentrate or exclude all-zero (baseline) periods.
- **Temporal order:** Preserve row order throughout. No shuffling before or during validation.
- **Subset strategy:** If the full 4.18M row dataset is too large for a given experiment, subsetting must be done by temporal slicing — not random sampling — and the zero/nonzero balance must be verified after subsetting.
- **Scale difference:** `methane_ppm` and `ethylene_ppm` operate on different scales (0–297 vs 0–20). Separate scalers or per-target evaluation metrics should be used.

---

## Verdict

- [x] Dataset is fit for EDA — proceed to Step 2 (pending user approval)

**Notes:** EDA must account for the high proportion of zero-target rows. Plots and correlation analyses should distinguish zero-gas (baseline) and active-gas periods where relevant. Do not proceed to feature engineering without a separate approval step after EDA.
