# Week 3 & Week 4 — Task Audit and Execution Plan

**Date:** 2026-05-28
**Project:** Fresh regression benchmark — methane_ppm and ethylene_ppm from 16-sensor gas array.
**Source plan:** `docs/Jibril_One_Month_Plan.pdf`.
**Inputs:** Completed Stages 06–11 + Week 1 artifacts (`reports/week1_fairness_memo.md`, `reports/week1_table_audit.md`).
**Status:** Plan for review. No code or notebooks generated yet.

---

## Part A — Task-by-task audit

### Week 3 — Coreset selection and creative direction

---

#### Task 3.1 — Enter the coreset-selection direction

**What the plan requires.** Study coreset selection as a principled alternative to arbitrary subsampling. Implement and understand at minimum four strategies: random coreset, k-means prototype selection, farthest-point sampling, density-aware sampling. The aim is not yet to compare them on the regression task (that is Task 3.2) but to establish the methodology and produce the coreset artifacts at the same budget as Stage 05.

**Status:** **NOT STARTED.**

**What Stages 06–11 contribute.**
- Stage 05 produced a stratified-by-regime subset, which is a valid baseline ("smart random") for Task 3.2's comparison but is not itself a coreset method in the principled sense.
- The fair-subset framework — fixed budget, temporal-order preservation, monotonic index artifact, fold-intersection at use site — is reusable. Each new coreset can be saved in the same parquet schema and dropped into the existing pipeline with no Stage 06–11 code changes.
- `validation_splits.parquet` is unchanged; coreset experiments use the same temporal folds.

**What is missing.**
- Implementations of: random coreset, k-means prototype selection, farthest-point sampling, density-aware sampling.
- Per-strategy subset artifacts at the 1M-row budget (one parquet each, same schema as `fair_subset_indices.parquet`).
- A figure showing coverage of the 4-way regime balance and temporal density for each strategy (matches Stage 05's diagnostic style).
- A short methodology note explaining each strategy and the rationale for the parameter choices (k for k-means, neighborhood radius for density-aware, etc.).

**Minimum work needed.**
A single notebook `12_coreset_strategies.ipynb` that:
1. Loads the full processed dataset.
2. Implements each of the 4 strategies as a function taking `(X, y, budget) -> sorted_row_indices`.
3. Runs each at budget = 1,000,000 rows (matching Stage 05 for direct comparability).
4. Saves four parquet artifacts: `data/subsets/coreset_random.parquet`, `coreset_kmeans.parquet`, `coreset_farthest.parquet`, `coreset_density.parquet`. Same schema as Stage 05.
5. Produces one diagnostic figure (regime balance + temporal coverage per strategy), saved to `results/figures/12_coreset_strategies.png`.
6. Saves a memo `results/memos/12_coreset_strategies.md` with method descriptions and parameter justifications.

**Implementation notes.**
- K-means: use sklearn's MiniBatchKMeans with k = budget (1M centers is infeasible). The practical approach is k-means at a small k (e.g., 1000 clusters), then sample within each cluster proportionally — a "stratified-by-cluster" coreset. Alternatively, use k-means++ initialization at k=budget as approximate prototypes. The choice should be justified in the memo.
- Farthest-point sampling on 4M rows is expensive. A scalable variant ("greedy k-center" with submodular speedups, or random-projection accelerated FPS) is required. Default to a subsampled candidate pool (e.g., FPS over a 10M random pre-sample, but here the dataset is 4M so the full set is the pool).
- Density-aware: use kernel-density estimation on the 16-dim sensor space (or on PCA-reduced sensors) and sample inversely proportional to density to upweight rare regions.
- Coreset operations must preserve temporal sortability — index lists are always sorted before saving, matching the Stage 05 convention.

---

#### Task 3.2 — Compare coreset strategies on the regression task

**What the plan requires.** Take the strategies from Task 3.1 and train the same leading model(s) on each. Compare results under the existing temporal validation protocol. Determine which strategy yields the best regression quality at the fixed budget and whether some models are more sensitive than others to subset selection.

**Status:** **NOT STARTED.**

**What Stages 06–11 contribute.**
- The full 5-fold temporal validation harness is reusable: `prepare_fold()` in Stages 06–11 only needs the subset parquet path swapped.
- Stage 11 produced the tuned configs for `spline_tuned` and `extratrees_tuned`, which are the recommended reference models for this task (see Part B below).
- The metric-recording and aggregation utilities from Stage 11 carry over directly.

**What is missing.**
- A re-evaluation of the two reference models on each coreset strategy (4 strategies × 2 models × 5 folds × 2 targets = 80 metric records) plus the Stage 05 stratified baseline as a 5th strategy.
- A sensitivity table: per-model, per-strategy R² with mean ± std.
- A short recommendation of the best subset-selection mechanism, supported by the table.

**Minimum work needed.**
A single notebook `13_coreset_comparison.ipynb` that:
1. Loads each coreset artifact from Task 3.1 plus the Stage 05 stratified baseline.
2. For each (strategy × model) combination, runs the full 5-fold protocol with the model's best config from Stage 11 frozen — no re-tuning.
3. Saves `results/tables/13_coreset_metrics_long.parquet` and a summary table.
4. Produces `results/figures/13_coreset_comparison.png` (per-model R² bar chart grouped by strategy).
5. Saves a memo `results/memos/13_coreset_comparison.md` with the recommendation.

**Implementation notes.**
- No new tuning. The Stage 11 best configs are held fixed so the only variable is the subset.
- Compute budget: 5 strategies × 2 models × 5 folds. With frozen configs this is approximately the cost of a fresh Stage 11 best-config refit per strategy. Estimated ~2 hours for spline, ~3 hours for extratrees → ~5 hours total.
- The Stage 05 stratified subset must be re-evaluated under this protocol (not just copied from Stage 11) so that all five strategies are compared on equal footing.

---

#### Task 3.3 — Propose a first creative model direction

**What the plan requires.** Based on the comparative study (Stages 06–11) and the coreset results (Tasks 3.1–3.2), propose one original direction. Acceptable archetypes per the PDF: (a) coreset-aware spline model, (b) spline + boosting residual model trained on a coreset, (c) regime-aware model separating low- and high-concentration behavior, (d) teacher-student compression pipeline. The output is a written proposal, not yet an implementation (that is Task 4.1).

**Status:** **NOT STARTED.** The benchmark phase ended at Stage 11; no creative-direction proposal exists in `reports/` or `results/memos/`.

**What Stages 06–11 contribute.**
- The diagnostic evidence the proposal must reference: smooth-function ceiling at R² ≈ 0.62, ethylene's hard cap at R² ≈ 0.58, fold-1 ethylene as a structural problem, partition-based methods need heavy regularization, neural methods underperform smooth methods.
- The Stage 11 winners (spline_tuned and poly_tuned) point toward a spline-centric direction.
- The per-fold breakdowns motivate a regime-aware approach: failures concentrate where the val regime departs from train, suggesting an explicit regime classifier could route inputs to specialist sub-models.
- The huber result (best MAE despite 4th-rank R²) suggests a robust-loss component is valuable.

**What is missing.**
- A written proposal selecting one of the four archetypes (or a defensible hybrid).
- A statement of which diagnostic evidence from Stages 06–11 motivates the choice.
- A statement of what success looks like (concrete target metric or comparison condition).
- A rough sketch of the prototype's components so Task 4.1 can implement directly.

**Minimum work needed.**
A short memo `reports/week3_creative_direction_proposal.md` that:
1. Recommends one archetype with a one-paragraph justification grounded in Stages 06–11 evidence.
2. Lists the prototype's components: backbone model, subset-selection mechanism, residual or compression layer, regime split (if used).
3. Specifies the comparison condition: same fair-subset budget, same 5-fold protocol, same metrics; baseline = `spline_tuned` from Stage 11.
4. Defines a "promising" threshold (e.g., "improves pooled R² by ≥0.01 *and* reduces fold-1 ethylene collapse by ≥0.05") so Task 4.2's interpretation is grounded.

**Recommended archetype (working hypothesis, pending diagnostic from 3.2).** A *coreset-aware spline + boosting residual* model: spline (the proven backbone) trained on a coreset-selected subset, with a small gradient-boosting model fitted on the spline residuals. This is the archetype the Stage 11 diagnostics most directly support — spline supplies the smooth global fit, boosting handles structured local error patterns, and the coreset addresses the fairness constraint. A regime-aware variant (separate models for low- and high-concentration regimes) can be a fallback if the residual is too noisy to model.

---

#### Task 3.4 — Explain transferability to classification

**What the plan requires.** For each regression model studied (especially XGBoost, LightGBM, CatBoost, ExtraTrees, HistGradientBoosting — all of which also classify), produce a short note identifying which mechanisms, behaviors, or design choices may transfer to Otmane's classification branch. Coordinate with Otmane so the two branches converge. No classification experiments required.

**Status:** **NOT STARTED.**

**What Stages 06–11 contribute.**
- Per-model behavior under temporal drift (fold variance, fold-1 vulnerabilities) — directly transferable observations.
- The diagnostic that boosted methods underfit at 100 iterations is a general finding, not regression-specific.
- The diagnostic that sklearn's HGBR built-in early stopping fails on temporal data — directly transferable to any sklearn classifier using the same mechanism.
- The diagnostic that heavy regularization helps tree methods generalize across temporal folds — directly transferable.

**What is missing.**
- A short note (1–2 pages) per the PDF. Should be structured as: model family → mechanism observed in regression → expected transfer to classification → suggested coordination point with Otmane.
- An action: send the note to Otmane and capture his feedback, or schedule a 30-minute alignment meeting.

**Minimum work needed.**
A short memo `reports/week3_transferability_note.md` that:
1. Lists the five tree/boosted families studied in Stages 09 and 11.
2. For each: one paragraph on the regression diagnostic and its classification implication.
3. One closing paragraph on coordination logistics with Otmane (what to align on, what data both branches should share, whether the same fair_subset is reusable for classification).

---

### Week 4 — Prototype and paper structure

---

#### Task 4.1 — Implement first prototype

**What the plan requires.** A single executable prototype of the direction chosen in Task 3.3. Comparison against the best fair baseline (Stage 11 best). Same sample size, same rolling temporal validation. The prototype does not need to be final — it must run, produce metrics, and be compared.

**Status:** **NOT STARTED.** Depends on Tasks 3.1–3.3.

**What Stages 06–11 contribute.**
- The full evaluation harness (`prepare_fold`, metric recording, aggregation, leaderboard scaffolding) is directly reusable.
- The baseline for comparison is the saved `spline_tuned` artifact in `results/models/tuned_11/fold_{k}_spline_tuned.joblib`.
- The Stage 11 search log can identify whether the prototype's hyperparameters are inside or outside previously explored ranges.

**What is missing.**
- The prototype implementation itself, contingent on the Task 3.3 archetype choice.
- A side-by-side metric table: prototype vs spline_tuned, same protocol, same metrics, per fold and per target.

**Minimum work needed.**
A single notebook `14_prototype_v1.ipynb` that implements the Task 3.3 archetype, evaluates on the same 5 folds with the chosen coreset (recommended: the Task 3.2 winning strategy), and saves:
- `results/tables/14_prototype_metrics_long.parquet` and summary.
- `results/figures/14_prototype_comparison.png`.
- `results/memos/14_prototype_v1.md` with a comparison vs `spline_tuned`.

**Implementation notes.**
- If the chosen archetype is *spline + boosting residual on a coreset*: the spline fits first (same config as `spline_tuned`), residuals are extracted on the training set, a small HGBR or LightGBM with strong regularization is fit on the residuals, and the final prediction is spline + residual_model. Implementation is straightforward — both components already exist as sklearn-compatible estimators.
- If the chosen archetype is *regime-aware*: implement a regime classifier (binary on `methane>0`) using the same fair-subset training data, then fit separate `spline_tuned` per regime. This is a 3-model pipeline (classifier + 2 specialists).
- The prototype must reuse the SEED=42 / RobustScaler / same-fold protocol verbatim. Fairness inherits from Week 1.

---

#### Task 4.2 — Interpret the prototype scientifically

**What the plan requires.** Analyze whether any improvement comes from the subset-selection mechanism, the nonlinear backbone, the residual correction, the regime split, or the compression idea — i.e., identify which ingredient is doing the work. Not just "the prototype is better/worse," but *why*.

**Status:** **NOT STARTED.** Depends on Task 4.1.

**What Stages 06–11 contribute.**
- The Stage 11 fold-level metrics provide the baseline to attribute prototype gains against.
- Stage 11's spline_tuned per-fold breakdowns let the interpretation isolate fold-specific gains (e.g., "the residual model helps on fold 1 ethylene but not fold 3").
- The Task 3.2 coreset comparison provides the subset-attribution channel: if the prototype's gain comes from the coreset, swapping the coreset out should remove the gain.

**What is missing.**
- An ablation: prototype components removed one at a time, measured on the same folds.
- A written interpretation in the prototype memo (Task 4.1's `14_prototype_v1.md`) or a separate analysis memo.

**Minimum work needed.**
Extend `14_prototype_v1.ipynb` with an ablation block:
1. Full prototype (backbone + residual + coreset).
2. Backbone only (= `spline_tuned`).
3. Backbone + residual on Stage 05 stratified subset (residual contribution).
4. Backbone + residual on Task 3.2 winning coreset (coreset contribution).
5. (Optional) Backbone on the winning coreset, no residual (coreset alone).

Save an ablation table and a one-paragraph interpretation per ablation row.

---

#### Task 4.3 — Repair paper structure

**What the plan requires.** Reintegrate added material so that Section 4 (Experimental Results and Discussion) remains the single main experimental section. Baseline comparison, fairness protocol, coreset analysis, and tuned-model evaluation should appear as coherent subsections inside Section 4 — not as separate top-level Sections 5 and 6.

**Status:** **NOT STARTED.** The fresh project does not yet contain a paper draft in `reports/`. The PDF's reference to "Sections 5 and 6" pertains to the archived draft.

**What Stages 06–11 contribute.**
- The Stage 06–11 memos in `results/memos/` are subsection-sized and can be repurposed as paper text with minor editing.
- The Week 1 fairness memo and table audit (just written) are paper-grade subsection drafts.
- The Stage 11 cross-stage leaderboard is the canonical results table.

**What is missing.**
- An outline of the planned Section 4 structure with subsection headings.
- The decision of whether to write a fresh Section 4 or import and reorganize the archived draft. The PDF implies importing and reorganizing.

**Minimum work needed.**
A single outline document `reports/week4_paper_structure.md` containing:
1. Section 4 outline with subsection numbering (4.1 Protocol, 4.2 Baseline comparison, 4.3 Coreset analysis, 4.4 Tuned-model evaluation, 4.5 Prototype).
2. For each subsection: which existing memo/artifact supplies the content.
3. Explicit removal of the old Sections 5 and 6 — i.e., the directive that nothing should live at the top level outside Section 4 unless it is the new conclusions/discussion section.

If the supervisor wants the paper itself rewritten, that becomes a separate writing task tracked in the Task 4.4 closing report.

---

#### Task 4.4 — Two-part closing note

**What the plan requires.** A short internal report with two parts: (1) what was learned scientifically during the fair-comparison and coreset study, (2) what the next-month plan should be to transform the prototype into a deployable model for Pergamon.

**Status:** **NOT STARTED.** Depends on Tasks 3.1–4.3.

**What Stages 06–11 contribute.**
- The full set of scientific findings (Section 6 of Stage 11's assessment, the stable findings in the Week 1 table audit).
- The Stage 11 leaderboard, search log, and per-fold breakdowns.
- The deployment-relevant observations: smooth methods are cheapest at inference, neural methods are heaviest, ExtraTrees' regularized config is compact (200 small trees).

**What is missing.**
- Synthesis of the coreset results (Task 3.2) and the prototype interpretation (Task 4.2) into deployment-facing recommendations.
- A concrete next-month roadmap with milestones.

**Minimum work needed.**
A short report `reports/week4_closing_report.md` with the two parts specified by the plan, plus an appendix listing all artifacts produced over the month (notebooks, memos, parquets, figures, models). Length target: 4–6 pages.

---

## Part B — Why `spline_tuned` and `extratrees_tuned` as reference models for Week 3?

The Task 3.2 coreset comparison must use a small, well-justified set of reference models. Running every Stage 11 model on every coreset is wasteful (50 model-strategy combinations) and dilutes the diagnostic signal. Two reference models is the right number, and these two are the right pair.

### Selection criteria

A reference model for coreset comparison should be: (a) one of the best in its family after Stage 11 tuning, (b) stable enough that coreset-induced variance is observable above baseline noise, (c) representative of a distinct modeling regime, and (d) cheap enough that running it across 5 strategies × 5 folds is feasible.

### Why `spline_tuned`

- **Benchmark winner.** R² = 0.619 pooled, methane R² = 0.661 (#1), ethylene R² = 0.578 (tied #1). It is the model the prototype will most likely be built on.
- **Most stable.** Methane R² std = 0.042 (tied for lowest in the benchmark). Any coreset-induced degradation will stand out cleanly against this baseline noise.
- **Closed-form fit on a fixed basis.** Once the basis is computed, Ridge solves in milliseconds. spline_tuned is essentially a *direct* estimator from the data, so it amplifies the signal in the training distribution. **If a coreset is good, spline should reveal it.**
- **Cheap.** Per-fold fit time is seconds even with 5-knot cubic + Ridge head on 395k rows. The full Task 3.2 grid for spline runs in under 30 minutes.
- **Representative of the "smooth global function" regime** that dominates the benchmark.

### Why `extratrees_tuned`

- **Best non-smooth model.** R² = 0.567 pooled, methane R² = 0.626 (#4), with the largest absolute tuning gain in the benchmark (+0.132 over untuned).
- **Equally stable.** Methane R² std = 0.042 — the same low std as spline. Stability is matched, which means the *only* meaningful difference between the two reference models is their inductive bias.
- **Data-hungry by design.** With 200 trees each capable of depth 10, the model needs sample diversity per partition to estimate leaf statistics. **If a coreset is bad, extratrees should reveal it** through degraded fold variance or fold-1-style collapses.
- **Cheap-ish.** Per-fold fit time is ~20–30 seconds. Full Task 3.2 grid runs in roughly an hour.
- **Representative of the "partition-based ensemble" regime** that the benchmark places below the smooth ceiling but well above all boosted/neural alternatives.

### How the pair functions as a diagnostic

The two models bracket the model space along the axis the plan cares about: smooth-global vs partition-local inductive bias. They have opposite data appetites — spline benefits most from *representative* samples, extratrees benefits from *diverse* samples — which means they respond differently to coreset quality:

| Outcome of Task 3.2 | Interpretation |
|---|---|
| Spline improves, ExtraTrees unchanged | Coreset selects a more representative training distribution; smooth-function variance shrinks. Recommend the coreset for smooth-method deployment. |
| Spline unchanged, ExtraTrees improves | Coreset selects a more *diverse* training distribution; partition-method splits are better-informed. Recommend the coreset for tree-based deployment. |
| Both improve | Coreset is genuinely better than stratified-random for this problem. Recommend universally; use it for the prototype. |
| Both degrade | Coreset destroys signal (e.g., farthest-point picks edge cases, k-means picks cluster centers far from typical operating regimes). Reject and document the failure mode. |
| Both unchanged | The 1M-row budget is already saturating the fold protocol's distinguishing power. Either reduce the budget (e.g., 500k, 250k) to surface differences, or accept that the Stage 05 stratified baseline is already near-optimal at this size. |

Each outcome leads to a different actionable recommendation, which is precisely what Task 3.2 asks for ("determine whether some models are especially sensitive to the quality of the selected subset").

### What is *not* used as a reference

- **Boosted models (LGBM, CatBoost, HGBR)** — already known to underperform Ridge by ≥0.09 after exhaustive tuning. Their behavior on coresets is a secondary diagnostic; including them would triple the compute without changing the central conclusion.
- **TabNet and MLP** — too expensive (TabNet ~3.5h per full sweep, MLP ~3h) and too noisy across folds to give a clean coreset signal.
- **Poly_tuned** — strongly correlated with spline_tuned in behavior (both are smooth basis + Ridge). Adds redundancy without a new diagnostic dimension.
- **Ridge_tuned** — a special case of poly_tuned at degree 1. Same redundancy issue.

If the Task 3.2 results are inconclusive, adding one boosted model (CatBoost as the most fold-stable boosted) as a third reference is a reasonable extension. This should be a decision made after seeing Task 3.2's primary table, not committed up front.

---

## Part C — Execution plan and roadmap

**Sequencing constraint.** Every Week 4 task depends on Week 3 artifacts. Within Week 3, Task 3.2 depends on 3.1, Task 3.3 depends on 3.2's results. Task 3.4 is independent and can be written in parallel.

**Proposed order:**

| step | artifact | depends on | est. effort | est. compute |
|---|---|---|---|---|
| W3-A | `notebooks/12_coreset_strategies.ipynb` (Task 3.1) | Stages 03, 05 | 1 day implementation | ~30 min execution |
| W3-B | `notebooks/13_coreset_comparison.ipynb` (Task 3.2) | W3-A | 0.5 day | ~5 hours execution |
| W3-C | `reports/week3_creative_direction_proposal.md` (Task 3.3) | W3-B | 0.5 day writing | — |
| W3-D | `reports/week3_transferability_note.md` (Task 3.4) | Stages 09, 11 (independent of W3-A/B/C) | 0.5 day writing | — |
| W4-A | `notebooks/14_prototype_v1.ipynb` (Tasks 4.1 + 4.2) | W3-C | 1 day implementation + ablation | depends on archetype, ~2–4 hours execution |
| W4-B | `reports/week4_paper_structure.md` (Task 4.3) | All memos and Week 1 artifacts | 0.5 day writing | — |
| W4-C | `reports/week4_closing_report.md` (Task 4.4) | W3-A through W4-B | 1 day writing | — |

**Total wall-clock estimate:** ~4.5 working days of implementation + writing + 8 hours of compute spread across W3-B and W4-A.

**Critical-path note.** W3-A → W3-B → W3-C → W4-A → W4-C is the critical path. W3-D and W4-B can be written in parallel with the compute-heavy steps.

**Decision points the supervisor should weigh in on:**
1. Whether the recommended archetype (coreset-aware spline + boosting residual) is the right Task 3.3 choice, or whether one of the other three archetypes is preferred.
2. Whether the 1M-row budget should be held fixed for Week 3 or whether a budget-sweep (e.g., {250k, 500k, 1M}) should be added to surface coreset-vs-stratified differences.
3. Whether Task 4.3 should produce only an outline (this plan's minimum) or a fully reorganized draft (significantly more writing time).
4. Whether a third reference model (CatBoost) should be added to Task 3.2 as a contingency.

**Out of scope for this month** (deferred to the Task 4.4 next-month roadmap):
- Multi-seed evaluation for stochastic models.
- Train/val gap recording for overfit diagnosis.
- Inference time and model size profiling (closes Task 2.4's partial status).
- Replacing HGBR's built-in early stopping with a manual temporal carve.
- Any deployment-engineering work (serialization for production, ONNX export, latency benchmarks).

---

**Awaiting approval before generating any notebooks or code.**
