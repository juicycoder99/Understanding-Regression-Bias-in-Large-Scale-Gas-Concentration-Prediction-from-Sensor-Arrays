# 02 — EDA Summary

**Date:** 2026-05-18
**Dataset:** data/processed/ethylene_methane.parquet

---

## Gas Activity Breakdown

- Rows with `methane_ppm`  > 0 : 1,849,752  (44.3%)
- Rows with `ethylene_ppm` > 0 : 1,802,960  (43.1%)
- Rows with either target > 0 : 2,804,114  (67.1%)
- Rows with both targets > 0 : 848,598  (20.3%)
- **Pure baseline rows (both targets = 0): ~32.9%**

---

## Sensor–Target Correlations (Pearson, all rows)

**Top 5 for `methane_ppm`:**
- s06: r = 0.758
- s13: r = 0.746
- s14: r = 0.741
- s05: r = 0.738
- s04: r = 0.419

**Top 5 for `ethylene_ppm`:**
- s15: r = 0.653
- s16: r = 0.612
- s11: r = 0.600
- s09: r = 0.531
- s07: r = 0.527

---

## Sensor Clustering (|r| > 0.95)

The 16 sensors collapse into **~6 near-redundant groups**:

| Group | Sensors | Behaviour |
|-------|---------|-----------|
| A | s03, s04 | r = 0.999 — essentially identical |
| B | s05, s06, s13, s14 | r ≥ 0.996 — methane-responsive cluster |
| C | s07, s08, s12 | r ≥ 0.993 — broad sensitivity |
| D | s09, s10 | r = 0.996 |
| E | s11, s15, s16 | r ≥ 0.985 — ethylene-responsive cluster |
| F | s01 | mid-correlation with D (r ≈ 0.97) |
| — | s02 | **independent** (≈ 0 correlation with everything, including targets) |

**Implication:** 16 raw sensors carry roughly the information of 5–6 independent channels.

---

## Cross-Sensitivity (Active-Gas Rows Only)

When restricted to active-gas rows, sensor–target correlations reverse sign in revealing ways:

- **Methane cluster (s05, s06, s13, s14):** +0.74–0.77 with methane, **−0.47 to −0.51 with ethylene**.
- **Ethylene cluster (s11, s15, s16, s09, s10):** +0.52–0.60 with ethylene, **−0.13 to −0.39 with methane**.
- **s02:** Remains uncorrelated with both targets in all regimes — possibly a reference/temperature/humidity channel.

This is the signature of competitive cross-sensitivity in a metal-oxide sensor array: when one gas dominates, sensors tuned for the other gas show suppressed response. The two targets cannot be modelled independently without information leakage.

---

## Target Distribution Behaviour

- Heavy **zero-inflation** (~32.9% pure baseline rows).
- Both targets are right-skewed when restricted to nonzero values.
- `methane_ppm` operates 0–296.67 ppm; `ethylene_ppm` operates 0–20.00 ppm — **~15× scale difference**.
- Targets are step-like over time (chamber concentration switches between fixed levels) — not smoothly varying.

---

## Baseline vs Active-Gas Sensor Means

Bar chart confirms a clear shift in mean sensor value between baseline and active-gas regimes for most sensors. `s02` is the only sensor that does not respond — reinforcing its likely role as a reference channel.

---

## Figures Saved

- `results/figures/02_target_distributions.png`
- `results/figures/02_target_timeseries.png`
- `results/figures/02_regime_map.png`
- `results/figures/02_sensor_timeseries.png`
- `results/figures/02_sensor_boxplot.png`
- `results/figures/02_sensor_target_correlation.png`
- `results/figures/02_sensor_sensor_correlation.png`
- `results/figures/02_sensor_response_baseline_vs_active.png`

---

## Implications for Preprocessing & Feature Engineering

1. **Scaling is mandatory.** Targets are on different scales (300 vs 20), and sensor magnitudes likely differ as well. Use `StandardScaler` fit on **training folds only** (no leakage into validation).
2. **Multicollinearity is severe.** Tree-based models (RandomForest, GradientBoosting, XGBoost) tolerate it; linear models (Ridge, Lasso, ElasticNet) will be unstable on raw 16 features unless regularised.
3. **Do not drop redundant sensors before benchmarking.** The redundancy itself is a property worth measuring — fair benchmark should compare models on the **full 16-sensor feature set**. Dropping features now would bias the comparison.
4. **No lag/rolling/seasonality features yet.** CLAUDE.md rules forbid blind temporal features. Step-like target switches mean naive lags could leak future information through the chamber-protocol boundary.
5. **Multi-output regression.** Predict both targets jointly (or two single-output models with shared splits). Cross-sensitivity means per-target independent modelling will miss the joint structure.
6. **Zero-inflation matters for metrics.** Plain MSE will be dominated by the zero-baseline rows. Report metrics split by `baseline / active` regime in addition to overall.

---

## Verdict

- [x] **EDA complete.** Dataset is well-understood; no data-quality blockers.
- [x] **Proceed to Step 5: Preprocessing & Scaling.** Skip ad-hoc feature engineering — raw 16 sensors form the baseline feature set per fair-benchmark principle.
- [ ] Defer feature engineering (PCA, lag features, sensor-group selection) to **after** baseline benchmark, as comparison variants.

---

## Recommended Next Step

**Notebook `03_validation_split.ipynb`:** Build the rolling temporal validation protocol (no scaling/modelling yet). Verify each fold:
- Preserves temporal order.
- Has nonzero rows for both targets in both train and validation.
- Uses the same splits for every model in the benchmark.

This must be locked in **before** any preprocessing or modelling — it is the foundation of fair comparison.
