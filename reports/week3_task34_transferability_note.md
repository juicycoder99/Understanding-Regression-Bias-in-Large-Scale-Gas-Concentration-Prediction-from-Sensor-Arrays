# Task 3.4 — Transferability of Regression Findings to a Future Classification Pipeline

**Date:** 2026-05-29
**Project:** Fresh regression benchmark — methane_ppm and ethylene_ppm from 16-sensor gas array.
**Audience:** Jibril (regression branch) → Otmane (classification branch), for coordination.
**Status:** Evidence-driven draft for discussion with Otmane before finalizing. Closes Week 3, Task 3.4.
**Source evidence:** Stages 09 (untuned boosted), 11 (tuning), 12 (coreset construction), 13 (coreset comparison), 14 (prototype). Quantitative figures cited throughout are from the saved artifacts of those stages.

---

## 1. The question being addressed

> Given everything the regression study established (Stages 09–14), **which findings are likely to transfer to a future classification version of this gas-sensor task, which are regression-specific and unlikely to transfer, and what do the transferable findings imply for model selection, coreset selection, and temporal-validation/leakage control?**

The regression branch predicts continuous `methane_ppm` and `ethylene_ppm`. A classification branch would predict discrete labels (e.g., gas-present vs absent, or concentration-band classes). The two tasks share the **same 16 sensor inputs, the same temporal structure, and the same 4.18M-row source**, so mechanisms tied to the *data* (drift, sensor representativeness, temporal ordering) are candidates for transfer, while mechanisms tied to the *continuous loss* (R² ceilings, residual structure) are candidates for non-transfer.

No classification experiment has been run. Every forward-looking statement is an explicit, evidence-grounded **hypothesis**, not a result.

---

## 2. Findings that are likely transferable

Each item cites the quantitative regression evidence and states why the mechanism is data-driven (and therefore label-agnostic) rather than loss-driven.

### 2.1 Temporal distribution shift is the dominant failure mode (HIGH confidence)
**Evidence (Stage 11, 13).** Partition/boosted/neural models collapse on specific temporal folds. Fold-1 ethylene under tuned models: TabNet R² ≈ −0.00, LightGBM 0.06, CatBoost 0.21 — versus 0.55–0.59 for smooth models on the same fold. CatBoost methane fold-std after tuning = 0.053; HGBR methane fold-std = 0.358 (a model that swings from R² −0.21 to +0.68 across folds).
**Why it transfers.** Drift is a property of the sensor stream over time, independent of whether the label is continuous or categorical. The same fold-1 covariate shift that destroys regression R² will shift class-conditional feature densities.
**Transfer:** strong. Expect classifier accuracy/F1 to show the same per-fold instability.

### 2.2 Boosted models underfit at default iteration counts (HIGH confidence)
**Evidence (Stages 09 → 11).** At 100 iterations: LightGBM R² 0.327, CatBoost 0.353. At 2000 iterations + early stopping: LightGBM 0.499 (+0.172), CatBoost 0.514 (+0.161). The gain is almost entirely iteration count.
**Why it transfers.** Iteration count vs underfitting is a property of the boosting optimization, not the loss family. A cross-entropy objective underfits at 100 rounds for the same reason a squared-error objective does.
**Transfer:** strong. Expect a comparable accuracy jump from raising `n_estimators` to ~2000 with early stopping.

### 2.3 Heavy regularization is required for temporal generalization (HIGH confidence)
**Evidence (Stage 11 winning configs).** Every improved tree/boosted model won at the regularized end: ExtraTrees `max_depth=10` (vs unlimited) + `min_samples_leaf=50` (vs 20) drove the benchmark's largest tuning gain (0.435 → 0.567, +0.132); LightGBM won at `max_depth=3`; CatBoost at `l2_leaf_reg=10`. Stage 14 reinforced this: the *most* stable tree residual (ExtraTrees, neutral) outperformed the *least* regularized residual behavior (CatBoost, harmful, −0.016).
**Why it transfers.** Regularization combats overfitting to temporal quirks in the feature space — again a data property. Shallow, high-leaf-minimum trees generalize across drift regardless of the output type.
**Transfer:** strong.

### 2.4 sklearn built-in early stopping is unsafe on temporal data (HIGH confidence, direct)
**Evidence (Stage 11).** HistGradientBoosting was the only model to **regress** under tuning: 0.325 → 0.291 (−0.034). Cause: `early_stopping=True` uses a non-temporal internal split, so stopping fires against the wrong distribution.
**Why it transfers.** `HistGradientBoostingClassifier` and `MLPClassifier` share the identical mechanism. This is a code-level fact, not a statistical one.
**Transfer:** direct (1:1). The classifier versions carry the same bug.

### 2.5 Diversity-based coresets (FPS) outperform representativeness-based coresets (HIGH confidence within the tested regime)
**Evidence (Stage 13).** FPS won **10 of 12** (model × budget) cells; k-means won 1, random 1. Best overall: FPS+spline@395k = 0.6221. FPS beat per-fold stratified in 11/12 cells.
**Why it transfers.** The coreset is constructed from **sensor features only** — it never sees the target. An FPS coreset is identical whether the downstream task is regression or classification. The *construction* is fully label-agnostic; only the downstream benefit must be re-confirmed.
**Transfer:** the artifacts transfer exactly; the magnitude of benefit must be re-measured for classification.

### 2.6 Partition models peak below full data budget (MODERATE confidence)
**Evidence (Stage 13).** CatBoost peaked at 25k rows/fold (0.5275) and *declined* with more data; ExtraTrees peaked at 60k (0.5834). Smooth models (spline) rose monotonically to 395k.
**Why it may transfer.** This reflects how tree partitions saturate on a drifting feature distribution — a data-geometry effect. Plausibly label-agnostic, but class imbalance could change the optimal budget, so confidence is moderate.
**Transfer:** likely, but budget-vs-accuracy curve must be re-derived for classification.

---

## 3. Findings that are regression-specific and may NOT transfer

These are tied to the continuous loss or the specific target geometry and should **not** be assumed for classification.

### 3.1 The R² ≈ 0.62 information ceiling
**Evidence (Stages 11–14).** No model across 2,350 tuned configs + 60 coreset variants exceeded R² ≈ 0.62. Stage 14 confirmed it is an information bound: a CatBoost stage that reaches R² 0.51 on raw targets extracted **zero** usable signal from spline residuals (V1 hybrid 0.606 < spline-only 0.622).
**Why it may not transfer.** R² measures continuous-variance explained. A classification task asking a coarser question (e.g., "is methane present?") may be **far more separable** than predicting exact ppm. The information needed to call a class boundary can be much less than the information needed to pin a continuous value. **Classification accuracy is not bounded by the regression R² ceiling.**
**Transfer:** do NOT carry the ceiling over. Classification may be substantially easier.

### 3.2 The methane-easier / ethylene-harder asymmetry
**Evidence (Stage 11).** Tuned methane R² reaches 0.66 (spline); ethylene caps at 0.58. CatBoost: methane 0.607 vs ethylene 0.421 — a 0.19 gap.
**Why it may not transfer.** The asymmetry is partly a *scale* effect (methane 0–300 ppm vs ethylene's narrower range) and partly a continuous-resolution effect. Under classification, the relevant question is class separability, which need not preserve this ordering — ethylene presence could be easier to classify than methane *level*.
**Transfer:** weak. Re-measure per-gas difficulty under the classification labels.

### 3.3 Spline / smooth-function dominance
**Evidence (Stages 11, 13).** Smooth global functions (spline, poly, ridge) topped every regression leaderboard; spline_tuned = 0.622 benchmark winner.
**Why it may not transfer.** Spline-on-Ridge is a *regression* construction (continuous basis expansion + linear head). Its classification analogue (e.g., spline features + logistic regression) is untested and may not dominate, because class boundaries can be sharp and local — precisely where smooth global functions are weakest and where tree/boosted methods (which lost in regression) may win.
**Transfer:** do NOT assume smooth methods win classification. This is the finding most likely to invert.

### 3.4 The residual-stage-is-noise result
**Evidence (Stage 14).** Spline + boosting residual failed all hypotheses (H1–H4); residuals were unpredictable (V1 0.606 < V0 0.622).
**Why it may not transfer.** This result is specific to the regression residual `y − spline(X)`. There is no direct classification analogue; a class-probability residual would be a different object.
**Transfer:** not applicable.

---

## 4. Expected implications for classification model selection

Grounded in the transfer analysis above, **not** in untested classification runs.

| candidate | regression evidence | classification expectation |
|---|---|---|
| **ExtraTrees** | best non-smooth model (0.567), most stable tree (methane std 0.042), largest tuning gain (+0.132) | **Strong first candidate.** Random-threshold drift resistance + low variance should transfer well. Likely ≥ Random Forest. |
| **CatBoost** | methane R² 0.607, most temporally stable boosted, but most coreset-sensitive (range 0.048) | **Strong candidate** if paired with a good (FPS) coreset; ordered boosting suits temporal order. Sensitive to subset quality. |
| **LightGBM** | methane R² 0.605, fastest, ethylene most variable (std 0.21) | **Good speed/accuracy candidate;** expect instability on the rarer/harder class. |
| **HistGradientBoosting** | regressed under tuning due to early-stopping bug | **Competitive only after the early-stopping fix.** Otherwise avoid. |
| **Smooth / linear (logistic, spline+logistic)** | dominated regression | **Unknown — do not presume dominance.** Test, but expect tree/boosted methods may win where regression smooth methods did. |
| **Random Forest (Otmane's existing)** | dominated by ExtraTrees at every budget | Expect ExtraTrees to be a stronger, equally cheap drop-in. |
| **Classical GradientBoosting (Otmane's existing)** | accurate but ~50× slower than histogram methods | Replace with HistGradientBoosting/LightGBM for accuracy-per-compute. |

**Net guidance:** the classification model ranking may **not** mirror the regression ranking. The safe transfers are the *design choices* (deep regularization, 2000 iterations, manual early stopping), not the *model ordering*. Tree/boosted methods that lost in regression could plausibly win in classification because class boundaries reward their local-partition bias.

---

## 5. Expected implications for coreset selection (FPS vs k-means vs others)

**Evidence recap (Stages 12–13).**
- Construction diagnostics (Stage 12): k-means best at *representativeness* (sensor-mean deviation 0.001, ~50× better than random); density worst (sensor-std inflation up to 4.62 at 25k); FPS highest *dispersion* (space-filling).
- Downstream model quality (Stage 13): **FPS won 10/12 cells**; k-means won 1; density was worst for CatBoost (last in all 4 budgets).

**Implications for classification:**
1. **Start with FPS.** It is label-agnostic and was the clear regression winner. The Stage 12 per-fold FPS artifacts can be reused directly.
2. **k-means representativeness did not convert to model wins in regression** — so do not assume the "most representative" coreset is best for classification either. Test FPS vs k-means head-to-head.
3. **Density-aware should be deprioritized.** It failed worst in regression and its sensor-distribution distortion (std inflation 4.62×) is label-independent — that distortion will carry into classification.
4. **Open question — class-stratified coreset.** None of the Stage 12 coresets balance *class* frequency, because they were built for regression. If the classification labels are imbalanced, a **class-stratified FPS variant** may be needed. This is the one coreset construction the regression study did not provide and the most likely gap for Otmane to fill.

---

## 6. Expected implications for temporal validation and leakage control

**Evidence recap.**
- Random k-fold would have hidden the fold-1 collapses entirely (Stage 11/13 per-fold spreads).
- Stage 12's per-fold leakage-clean construction pool and Stage 13's cross-protocol audit (all gaps within fold-std → Stage 05 leakage benign) established the discipline.

**Implications:**
1. **Adopt the 5-fold rolling temporal protocol** (`validation_splits.parquet`) for classification. Random k-fold accuracy on this dataset is almost certainly optimistic.
2. **Report per-fold spread, not just the mean.** The regression std across folds was large (e.g., HGBR methane std 0.358); classification F1 std is likely similarly large and is the honest measure.
3. **Fix early stopping first (§2.4).** This is a correctness prerequisite, not an optimization.
4. **Construct classification coresets per-fold from train ranges only**, mirroring Stage 12's Option-C decision, to keep the comparison leakage-clean and publishable.
5. **The cross-protocol audit method transfers** — Otmane can run the same "does the leakage-clean protocol differ from the convenient one within fold-std?" check on his classifiers.

---

## 7. Risks, assumptions, and limitations

1. **No classification experiment has been run.** Every section above is a hypothesis. The single highest-risk assumption is §3.3/§4 — that the model *ranking* transfers; it may invert.
2. **Different loss, different geometry.** Variance-control mechanisms validated under squared error may behave differently under cross-entropy. Confidence grades reflect this.
3. **Class definition is undefined.** "Classification" could mean gas-presence (likely easy), concentration-band (medium), or fine multi-class (hard). The transfer expectations change with the label granularity; §3.1 (no R² ceiling) assumes a coarser-than-regression target.
4. **Class imbalance is unaddressed.** All regression coresets are class-agnostic; imbalance could require new construction (stratified-by-class FPS) and changes the optimal-budget finding (§2.6).
5. **Single-seed evidence.** Regression used SEED=42 throughout; multi-seed variance was not characterized. Classification claims inherit this limitation.
6. **TabNet/MLP excluded from strong recommendations** — they underperformed in regression and are compute-heavy; their classification behavior is untested and not assumed.

---

## 8. Recommendations for Otmane's classification work

Ordered by priority and confidence:

1. **(Correctness, do first)** Audit every classifier using `early_stopping=True` on temporal data; replace the internal split with a manual temporally-latest 10–15% eval carve. (§2.4, direct transfer.)
2. **(Protocol)** Re-validate existing XGBoost / RF / GradientBoosting classifiers under the shared 5-fold rolling temporal protocol; report per-fold F1 mean ± std. Expect fold-1-style fragility to surface. (§2.1, §6.)
3. **(Quick win)** Raise boosted `n_estimators`/`iterations` to ~2000 with early stopping; expect a material accuracy jump. (§2.2.)
4. **(Tuning axis)** Grid the regularization axes (`max_depth`, `min_samples_leaf`/`min_child_samples`, `l2`/`reg_lambda`); expect optima at the regularized end. (§2.3.)
5. **(Coreset)** Reuse the Stage 12 per-fold FPS coresets as the fairness budget; benchmark FPS vs k-means vs random for classification. Build a **class-stratified FPS** variant if labels are imbalanced. (§5.)
6. **(Model set)** Prioritize ExtraTrees and CatBoost as candidates; treat smooth/linear methods as untested (do not presume they win as they did in regression). (§4.)
7. **(Audit)** Run the cross-protocol leakage check to confirm whichever subset protocol is chosen is benign. (§6.)

---

## 9. What we would expect to happen if the same principles were applied to a classification version of this dataset

This section is explicitly **speculative but evidence-anchored** — a set of falsifiable predictions, not results.

1. **Classification will likely be "easier" than regression in headline terms.** The R² ≈ 0.62 ceiling is a continuous-variance bound (§3.1). A coarse label such as gas-presence should be highly separable from 16 sensors; we would expect high accuracy / F1 (qualitatively, well above what R²=0.62 might naively suggest), because calling a boundary needs less information than pinning a value. **We do NOT predict a numeric accuracy** — only that it is not capped by the regression ceiling.

2. **The model ranking will likely re-order.** In regression, smooth methods won and tree/boosted methods trailed by 0.04–0.18. In classification, we would expect tree/boosted methods (ExtraTrees, CatBoost, LightGBM) to be **more competitive or dominant**, because class boundaries reward local partitioning — the exact bias that hurt these models on smooth continuous targets. The spline's regression advantage is the finding most likely to invert.

3. **Temporal drift will still hurt, and fold-1 will still be the weak fold.** Drift is label-agnostic (§2.1). We would expect classification F1 to dip on the same temporal folds where regression R² collapsed, with the same large fold-to-fold spread. Random k-fold would hide this.

4. **FPS coresets will likely remain the best fixed-budget selection,** with the caveat that class imbalance may demand a class-stratified FPS. We would expect FPS ≥ random/stratified for the majority-class regions and a possible need for class-balancing in minority regions.

5. **Boosted classifiers will likely peak below the full data budget,** mirroring the regression finding that CatBoost peaked at 25k and ExtraTrees at 60k — a deployment-relevant efficiency that may carry over, subject to imbalance effects.

6. **The methane/ethylene difficulty ordering may not hold.** Under classification, per-gas difficulty depends on class separability, not continuous resolution; ethylene (harder to regress) could be easy to detect as present/absent. We would expect the per-gas ordering to be **re-established empirically**, not inherited.

7. **The early-stopping bug will produce the same silent degradation** in any classifier that uses sklearn's built-in mechanism on temporal data — a direct, near-certain transfer (§2.4).

**Summary prediction.** If the same fairness, temporal-validation, regularization, and coreset principles are applied to a classification version, we would expect: (a) strong achievable accuracy unbounded by the regression ceiling, (b) a re-ordered model leaderboard favoring tree/boosted methods, (c) persistent temporal-drift fragility concentrated on the same folds, and (d) FPS coresets remaining valuable but possibly needing class stratification. The transferable assets are the **methodology and design choices**; the non-transferable assets are the **specific performance ceiling and model ranking**.

---

**Sign-off.** This note distinguishes data-driven (transferable) from loss-driven (non-transferable) findings using quantitative evidence from Stages 11–14, and frames all forward-looking statements as testable hypotheses. Conclusions are to be confirmed with Otmane before entering the paper. Highest-priority transfer: the early-stopping correctness fix (§2.4). Highest-risk assumption: that the model ranking transfers (§3.3, §9.2) — it likely does not.
