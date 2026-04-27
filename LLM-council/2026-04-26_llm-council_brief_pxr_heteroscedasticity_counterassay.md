# LLM-Council Brief — Closing the OpenADMET PXR Activity Leaderboard Gap

## Objective
Close the performance gap between the current project best submission (from local submissions log) and the current top-scoring submission on the public OpenADMET PXR activity leaderboard by:
1) tightening **absolute error** (MAE/RAE),
2) improving **rank ordering** (Spearman/Kendall), and
3) doing so in a way consistent with the dataset’s strong **heteroscedasticity** and with **judicious** use of counter-assay (PXR-null) data.

This document is formatted to be consumed by a multi-model advisory process (e.g., LLM Council) and to translate into concrete experiments.

## Sources
- Public leaderboard (activity): https://huggingface.co/spaces/openadmet/pxr-challenge/resolve/main/data/activity_leaderboard.csv
- Local submissions log: `activity/outputs/finalist_predictions/SUBMISSIONS_LOG.md`
- External uncertainty characterization writeup: https://raw.githubusercontent.com/quantumdolphin/OPENADMET-PXR-activity-pred/main/UNCERTAINTY_CHARACTERIZATION.md

## Current performance comparison (hard numbers)

### Project best (local)
From `activity/outputs/finalist_predictions/SUBMISSIONS_LOG.md`, best recorded submission is **Submission #12** (3-way prediction-level ensemble), with:
- RAE = **0.5878**
- MAE = **0.4681**
- R²  = **0.5626**
- Spearman ρ = **0.8070**
- Kendall τ  = **0.6081**

### Public top submission (current leaderboard rank #1)
From the public CSV, rank #1 `chemprop_enjoyer`:
- RAE_mean = **0.45**
- MAE_mean = **0.31**
- R2_mean  = **0.87**
- Spearman_R_mean = **0.91**
- Kendall’s_Tau_mean = **0.76**

### Gap to close (project best → #1)
- RAE: 0.5878 → 0.45  (Δ = **−0.1378**, ~**23%** relative reduction)
- MAE: 0.4681 → 0.31  (Δ = **−0.1581**, ~**34%** relative reduction)
- R²:  0.5626 → 0.87  (Δ = **+0.3074**)
- Spearman: 0.8070 → 0.91 (Δ = **+0.103**)
- Kendall:  0.6081 → 0.76 (Δ = **+0.152**)

Interpretation: the gap reflects both **missing signal / representation** and **suboptimal handling of noise + evaluation distribution**; it is unlikely to be fixed by weighting alone.

## Dataset reality: heteroscedasticity (why it matters)
The uncertainty characterization shows potency-dependent measurement noise:
- Strong negative correlation between potency and uncertainty:
  - Spearman corr(pEC50, stderr) ≈ **−0.850**
  - Spearman corr(pEC50, CI width) ≈ **−0.848**
- Heavy uncertainty tail:
  - median pEC50_std.error ≈ **0.15**
  - 95th percentile pEC50_std.error ≈ **0.574**
  - median CI width ≈ **0.58**
  - 95th percentile CI width ≈ **2.04**
- Weak potency decile can have median SE ≈ **0.57** and CI width ≈ **2.02** (vs strongest decile SE ≈ 0.0569, CI width ≈ 0.24).

Practical implication: the weak-activity regime is close to baseline / floor effects and yields poorly identified EC50 fits; treating these labels as equally reliable wastes model capacity and can degrade both MAE/RAE and rank metrics.

## Proposed modeling strategy
Two coupled thrusts:
1) **Heteroscedasticity-aware training** (make the model robust to the noisy weak tail without throwing away learnable SAR).
2) **Counter-assay incorporation** (use PXR-null signal as orthogonal information to reduce false positives / interference and improve ordering).

The intent is to improve *both* error and rank correlation, not just one.

---

# Part A — Specific heteroscedasticity-aware modeling strategies

## A1) Tuned total-error weighting (do not hardcode sigma_model)
Use per-row standard errors (SE) to weight training, but tune the model-error floor:

Weight form:
- w_i = 1 / (SE_i^2 + sigma_model^2)

Key implementation details:
- Treat `sigma_model` as a hyperparameter (not a constant). Suggested sweep:
  - sigma_model ∈ {0.10, 0.15, 0.25, 0.35, 0.50, 0.60, 0.80}
- Apply weight **clipping** to prevent a small set of very precise curves from dominating:
  - w_i ← clip(w_i, quantile_05(w), quantile_95(w))  (or fixed bounds)
- Prefer optimizing `sigma_model` for **RAE + Spearman** jointly (multi-objective) because leaderboard rewards both accuracy and ordering.

Why this helps:
- If sigma_model is too large, w_i becomes nearly uniform → heteroscedastic structure ignored.
- If sigma_model is too small, the model overfits the “clean” regime and becomes brittle to distribution shift.

## A2) Robust loss functions (reduce sensitivity to the weak/noisy tail)
Train with a robust objective so the model does not chase noisy residuals in the weak-activity regime.

Options:
- Pseudo-Huber loss (smooth L1) for boosting / NN.
- Quantile loss targeting median (and optionally additional quantiles for diagnostics).

Operational guidance:
- Evaluate (MAE, RAE, Spearman) stratified by uncertainty bins (high SE vs low SE). Robust losses should improve high-SE bins without collapsing correlations.

## A3) Heteroscedastic likelihood / mean-variance modeling (preferred when feasible)
Instead of only weighting, explicitly model aleatoric noise:
- Predict μ(x) and log σ²(x)
- Train with Gaussian NLL: L = 0.5 * exp(−logσ²) * (y−μ)^2 + 0.5*logσ²

Notes:
- You still submit μ(x) to the leaderboard.
- This tends to learn shrinkage toward plausible means in noisy regions, often improving MAE/RAE and stabilizing rank.

## A4) Regime-aware modeling (floor-effect handling)
Add explicit structure for the weak-activity tail:

Two-stage approach:
1) Classify whether compound is “weak/uncertain” vs “confident potency” (threshold defined on training using pEC50 and/or SE).
2) Fit separate regressors per regime or use a mixture-of-experts (MoE) gated by the classifier.

Lightweight variant (often sufficient):
- Add features derived from SE / CI width (or potency-bin priors) to allow the model to learn different bias/variance behavior.

## A5) Evaluation protocol aligned to heteroscedasticity
Every experiment should report not only global metrics but:
- MAE / RAE / Spearman stratified by:
  - pEC50 deciles, and
  - SE quantiles.

Decision criterion:
- Prefer changes that reduce error in high-SE bins **without** harming rank correlation in analog-rich regions.

---

# Part B — Judicious inclusion of counter-assay data (PXR-null)

## B0) Principle
Counter-assay is orthogonal signal: it is not always additive to the primary assay and should not be naively subtracted. The right use is to improve selectivity and reduce false positives driven by assay interference / non-specific transcriptional effects.

## B1) Use counter-assay as an auxiliary task (multi-task learning)
Train a shared representation with two heads:
- Head 1: main pEC50 (primary endpoint)
- Head 2: counter pEC50 (PXR-null)

Loss design:
- Mask missing counter labels.
- Apply heteroscedastic weighting on the main head (total-error or NLL).
- Use a reduced weight on counter loss (e.g., λ_counter ∈ {0.2, 0.5, 1.0}) tuned by validation.

Why this helps:
- Encourages the model to encode “general transcriptional activation / assay interference” patterns separately from true PXR activity.
- Improves ordering by penalizing compounds predicted high in both assays.

## B2) Add counter-derived features to tabular/XGBoost pipeline (fast path)
If the main production model is tabular + XGBoost/ensembles, incorporate counter data in a way that does not leak labels:

Feature construction options:
1) Train a counter-assay predictor on counter data only, generate **OOF predictions** for the main dataset, and use them as an additional feature.
2) Construct selectivity proxies:
   - predicted_selectivity = pred_main − pred_counter
   - or ratio-like transforms after calibration

Critical constraint:
- Use out-of-fold predictions for training rows (no target leakage).
- For test rows, use the counter predictor trained on all available counter data.

Why this helps:
- Trees can learn to downweight “likely interference” compounds and adjust rankings.

## B3) Regularize against counter-signal (soft constraint)
Impose a penalty that discourages high primary predictions when counter predictions are high:
- Add a term: L_penalty = α * ReLU(μ_counter(x) − t)^2 or α * μ_counter(x)
- Or post-hoc adjustment: y_adj = y_main − β * max(0, y_counter − c)

This is purposely soft: it should not destroy true actives that also show counter activity, but can improve ranking overall.

## B4) Calibration for ranking improvements
Because leaderboard ranking metrics are important (Spearman/Kendall):
- Apply monotone calibration (isotonic regression) on validation predictions for both main and counter heads.
- Then compute selectivity features from calibrated outputs to stabilize ordering.

---

# Integrated experimental plan (minimal set of high-value experiments)

## Experiment 1: Tune total-error weighting + weight clipping (tabular baseline)
- Base model: your strongest tabular/hybrid ensemble pipeline.
- Sweep sigma_model and weight clipping.
- Track: global MAE/RAE + Spearman, plus stratified metrics by SE quantile.

Success criterion:
- Meaningful RAE reduction without Spearman degradation; especially improved performance on high-SE bins.

## Experiment 2: Add counter-assay via OOF counter-pred feature (fast, safe)
- Train counter predictor → produce OOF counter predictions for training, predictions for test.
- Add as feature(s) to XGBoost.
- Re-tune sigma_model with counter features present.

Success criterion:
- Spearman/Kendall increase and MAE/RAE not worse; ideally improves both.

## Experiment 3: Multi-task model with heteroscedastic main loss
- Shared encoder, dual heads, masked counter loss.
- Main head uses total-error weights or NLL.
- Tune λ_counter.

Success criterion:
- Improves rank metrics and reduces error in uncertain regimes; adds complementary errors for ensembling.

## Experiment 4: Final ensemble weight search optimized for (RAE + Spearman)
- Instead of optimizing only RAE, optimize a composite objective:
  - J = RAE + γ * (1 − Spearman)
  - tune γ on validation.

Success criterion:
- Joint improvement of primary metric and ordering.

---

# Checklist: safeguards (avoid common failure modes)
- No leakage:
  - any counter-prediction features must be OOF on training.
- No extreme weights:
  - clip weights and/or enforce sigma_model floor.
- Verify improvement is not only on “easy” regime:
  - always report stratified metrics by SE/pEC50 bins.
- Validate against analog-generalization proxy:
  - ensure the split used for tuning resembles analog-heavy evaluation (avoid scaffold leakage).

---

# Summary: expected mechanism of gain
1) **Heteroscedastic-aware training** should reduce wasted capacity on noisy low-potency labels, improving MAE/RAE stability.
2) **Counter-assay integration** should reduce interference-driven false positives and improve ordering, boosting Spearman/Kendall.
3) Combining (1) and (2) in a controlled, leakage-safe way should move toward the leaderboard #1 regime where both absolute error and rank correlation are strong.
