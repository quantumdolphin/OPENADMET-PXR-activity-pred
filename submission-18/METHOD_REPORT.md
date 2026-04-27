# METHOD REPORT: Submission #18 (Ablation - Uncalibrated Super-Ensemble)

## Overview
Submission 17 (the Tier 3 Calibrated Super-Ensemble) demonstrated massive gains on the internal holdout split across all metrics (RAE dropped from 0.6000 to 0.5774), but completely degraded when evaluated on the public leaderboard (RAE degraded from 0.5878 back to 0.6057). 

This submission (#18) tests the primary hypothesis for the degradation: **Target Leakage / Distribution Mismatch via Isotonic Calibration.**

## Hypothesis
In Submission 17, we passed the blended prediction through an `IsotonicRegression` calibrator fit on the internal `holdout` split. Because Isotonic Regression is a non-parametric step function, it learned the exact structural "ledges" of the 414 holdout compounds. The public leaderboard evaluates on a fully blinded, structurally distinct distribution (analog expansion). The non-parametric ledges learned from the holdout split were fundamentally wrong for the test distribution, forcing continuous predictions into incorrect, discrete bins and destroying both rank ordering (Spearman) and absolute errors (RAE).

## Intervention
This submission is mathematically identical to Submission 17, with **one step removed**:
We took the raw linear blend of predictions using the identical optimal weights:
- w_15: 0.413 (Tier 1 XGBoost hybrid)
- w_8: 0.249 (Contrastive LF topology)
- w_11: 0.209 (RWPE End-to-end graph bias)
- w_16: 0.129 (Tier 2 MTL Neural Net - Aleatoric Uncertainty)

And we **skipped** the `IsotonicRegression.transform()` step entirely. The raw, continuous scalar blend `w_15*P_15 + w_8*P_8 + w_11*P_11 + w_16*P_16` is exported directly as the submission.

If the leaderboard performance of Submission 18 rebounds to >0.58 RAE and >0.80 Spearman, it definitively proves that the calibration step was the poison pill. If it remains degraded, the degradation is instead stemming from the base learners (e.g., counter-assay overfitting or NN out-of-distribution brittleness).