# Submission 18 (Ablation - Uncalibrated)

This directory contains Submission 18.

This is an ablation study of Submission 17. It uses the exact same 4-way super-ensemble blend and optimal weights, but skips the post-hoc `IsotonicRegression` calibration step. This is to test if the non-parametric calibrator overfit the internal holdout split and caused the observed leaderboard degradation.

See `METHOD_REPORT.md` for details.
