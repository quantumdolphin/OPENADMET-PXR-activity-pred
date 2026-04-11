# Uncertainty Characterization and Heteroscedasticity in the PXR Activity Dataset

## Confidence Interval Width and Standard Error

The PXR activity dataset provides both point estimates of compound potency (`pEC50`) and assay-derived uncertainty fields, including `pEC50_std.error`, `pEC50_ci.lower`, and `pEC50_ci.upper`. These values arise from nonlinear dose-response fitting and quantify how precisely the EC50 parameter is estimated for each compound. Dataset-level summaries are reported in [summary_metrics.csv](/home/avranga1008/pxr-challenge/activity/outputs/eda/target_quality/tables/summary_metrics.csv).

The standard error (SE) of `pEC50` reflects the sensitivity of the fitted potency estimate to experimental noise, replicate variability, and dose-response curve shape. The confidence interval (CI) can be written as

```text
CI = pEC50_hat +/- 1.96 * SE
```

and its width is

```text
CI width = pEC50_ci.upper - pEC50_ci.lower
```

which provides a direct uncertainty measure. Narrow intervals indicate well-constrained potency estimates, whereas wide intervals indicate poorly identified EC50 values. The strongest visual summaries of this behavior are [stderr_vs_pEC50.png](/home/avranga1008/pxr-challenge/activity/outputs/eda/target_quality/plots/stderr_vs_pEC50.png) and [ci_width_vs_pEC50.png](/home/avranga1008/pxr-challenge/activity/outputs/eda/target_quality/plots/ci_width_vs_pEC50.png).

## Observed Span of Uncertainty in the Dataset

The dataset contains 4,140 rows, with `pEC50` values spanning `1.61` to `7.55` and a median of `4.65` [summary_metrics.csv](/home/avranga1008/pxr-challenge/activity/outputs/eda/target_quality/tables/summary_metrics.csv). Uncertainty spans a broad range:

- Median `pEC50_std.error`: `0.15`
- 90th percentile `pEC50_std.error`: `0.418`
- 95th percentile `pEC50_std.error`: `0.574`
- Median CI width: `0.58`
- 90th percentile CI width: `1.561`
- 95th percentile CI width: `2.04`

At the low-uncertainty end, compounds have tight intervals and small SE values consistent with well-defined sigmoidal fits. At the high-uncertainty end, CI widths extend to roughly `2` log units and standard errors exceed `0.5`, consistent with noisy or weakly constrained curves. This spread is visible in [pEC50_interval_rank_plot.png](/home/avranga1008/pxr-challenge/activity/outputs/eda/target_quality/plots/pEC50_interval_rank_plot.png), [stderr_vs_pEC50.png](/home/avranga1008/pxr-challenge/activity/outputs/eda/target_quality/plots/stderr_vs_pEC50.png), and [ci_width_vs_pEC50.png](/home/avranga1008/pxr-challenge/activity/outputs/eda/target_quality/plots/ci_width_vs_pEC50.png).

## Relationship Between Potency and Uncertainty

A central empirical result is that uncertainty decreases monotonically as potency increases. The rank-based summary in [modeling_recommendations.csv](/home/avranga1008/pxr-challenge/activity/outputs/eda/target_quality/tables/modeling_recommendations.csv) reports strong negative Spearman correlations for both uncertainty metrics:

- `corr(pEC50, stderr) = -0.850`
- `corr(pEC50, CI width) = -0.848`

The decile summary makes the same pattern concrete [pEC50_decile_summary.csv](/home/avranga1008/pxr-challenge/activity/outputs/eda/target_quality/tables/pEC50_decile_summary.csv):

- In the weakest potency decile (`pEC50` `1.61-2.49`), median SE is `0.57` and median CI width is `2.02`
- In the strongest potency decile (`pEC50` `5.47-7.55`), median SE is `0.0569` and median CI width is `0.24`

This trend is directly visible in [stderr_vs_pEC50.png](/home/avranga1008/pxr-challenge/activity/outputs/eda/target_quality/plots/stderr_vs_pEC50.png), [ci_width_vs_pEC50.png](/home/avranga1008/pxr-challenge/activity/outputs/eda/target_quality/plots/ci_width_vs_pEC50.png), and the bin-level comparison in [stderr_boxplot_by_potency_bin.png](/home/avranga1008/pxr-challenge/activity/outputs/eda/target_quality/plots/stderr_boxplot_by_potency_bin.png).

## Origin of the Uncertainty Pattern

The observed potency-dependent uncertainty is consistent with expected assay and curve-fitting behavior:

1. Weak compounds generate responses closer to baseline, where signal is less separable from assay noise.
2. Shallow or poorly resolved dose-response curves produce a wider range of EC50 values compatible with the observed data.
3. Low-activity compounds are more susceptible to floor effects and reduced dynamic-range resolution.

The overall `pEC50` distribution supports this interpretation. [pEC50_histogram.png](/home/avranga1008/pxr-challenge/activity/outputs/eda/target_quality/plots/pEC50_histogram.png) and [pEC50_fine_histogram.png](/home/avranga1008/pxr-challenge/activity/outputs/eda/target_quality/plots/pEC50_fine_histogram.png) show a broad central potency mass with a long left tail into the weak-activity regime, where uncertainty is highest.

## Heteroscedasticity

These results indicate clear heteroscedasticity, meaning the variance of the measurement error is not constant across the potency range:

```text
Var(epsilon | pEC50) != constant
```

Instead, variance is highest for weak compounds and progressively smaller for potent compounds. The consequence is a fan-shaped uncertainty structure across the target range, visible in [stderr_vs_pEC50.png](/home/avranga1008/pxr-challenge/activity/outputs/eda/target_quality/plots/stderr_vs_pEC50.png) and [ci_width_vs_pEC50.png](/home/avranga1008/pxr-challenge/activity/outputs/eda/target_quality/plots/ci_width_vs_pEC50.png).

## Implications for Modeling

The heteroscedastic structure has direct modeling implications:

1. Equal-variance regression assumptions are violated.
2. Rows do not contribute equal information content.
3. Naive models may overfit low-potency, high-noise observations.
4. Structure-activity relationships can be blurred if noisy weak compounds are treated identically to precise potent compounds.

This is why [modeling_recommendations.csv](/home/avranga1008/pxr-challenge/activity/outputs/eda/target_quality/tables/modeling_recommendations.csv) recommends uncertainty-aware weighting rather than target binning or target transformation. A practical interpretation is that the dataset contains a higher-confidence regime at moderate to high `pEC50` and a lower-confidence regime in the weak-activity tail.

## Supporting Figures

The following PNG files directly support the claims in this section:

- [stderr_vs_pEC50.png](/home/avranga1008/pxr-challenge/activity/outputs/eda/target_quality/plots/stderr_vs_pEC50.png): standard error decreases sharply with increasing potency
- [ci_width_vs_pEC50.png](/home/avranga1008/pxr-challenge/activity/outputs/eda/target_quality/plots/ci_width_vs_pEC50.png): confidence interval width collapses as potency increases
- [stderr_boxplot_by_potency_bin.png](/home/avranga1008/pxr-challenge/activity/outputs/eda/target_quality/plots/stderr_boxplot_by_potency_bin.png): bin-level comparison of uncertainty across the potency range
- [pEC50_interval_rank_plot.png](/home/avranga1008/pxr-challenge/activity/outputs/eda/target_quality/plots/pEC50_interval_rank_plot.png): representative interval widths across compounds
- [pEC50_histogram.png](/home/avranga1008/pxr-challenge/activity/outputs/eda/target_quality/plots/pEC50_histogram.png): overall potency distribution
- [pEC50_fine_histogram.png](/home/avranga1008/pxr-challenge/activity/outputs/eda/target_quality/plots/pEC50_fine_histogram.png): finer view of low-end distribution structure

## Conclusion

The PXR activity dataset exhibits substantial, potency-dependent uncertainty. Standard errors and confidence interval widths are largest in the weak-activity regime and smallest among potent compounds, with near-monotonic decline across potency deciles. This is a clear heteroscedastic regression problem, not a uniform-noise potency series. Any downstream QSAR or ML model should account for that structure explicitly, preferably through uncertainty-aware weighting or other heteroscedastic modeling strategies.
