# Single-Concentration Signal and Statistics

This note explains the meaning of the signal and statistics columns in `pxr-challenge_single_concentration_TRAIN.csv`.

The short version is:

- `log2_fc_estimate` is the main biological signal.
- `log2_fc_stderr` describes uncertainty around that signal estimate.
- `t_statistic`, `p_value`, and `fdr_bh` describe how convincing the signal is statistically.
- `median_log2_fc`, `n_replicates`, and `cohens_d` help assess robustness and effect size.

## What Is Being Measured

This assay is a reporter-based screen for PXR activation. A compound is added to cells, and the assay measures how much the reporter signal changes relative to a baseline or negative-control condition.

The response is expressed as a log2 fold change:

- `0` means no change relative to baseline.
- `1` means a 2-fold increase in signal.
- `2` means a 4-fold increase in signal.
- `-1` means the signal is halved.

Because the scale is logarithmic, equal differences correspond to multiplicative changes in the raw signal.

## The Core Signal Columns

### `log2_fc_estimate`

This is the main assay readout for a row.

It is the estimated log2 fold change in reporter signal for that compound at that concentration, relative to the negative-control baseline.

Interpretation:

- Larger positive values suggest stronger activation in the assay.
- Values near `0` suggest little or no effect.
- Negative values suggest reduced signal relative to baseline.

Examples:

- `log2_fc_estimate = 1` means about a 2x increase in signal.
- `log2_fc_estimate = 0.5` means about a 1.41x increase.
- `log2_fc_estimate = 2` means about a 4x increase.

This is a single-concentration measurement, so it is not a potency estimate like `pEC50`. It only tells you the observed effect at the tested concentration on that plate/run.

### `median_log2_fc`

This is a more robust summary of the observed log2 fold change, based on the median rather than the model estimate alone.

Why it matters:

- The median is less sensitive to extreme values than the mean.
- It helps you see whether the estimated signal is supported by the underlying replicate-level behavior.

If `median_log2_fc` is close to `log2_fc_estimate`, that usually suggests the central tendency of the data agrees with the model-based estimate.

## The Uncertainty Column

### `log2_fc_stderr`

This is the standard error of `log2_fc_estimate`.

It describes how precisely the signal was estimated, not how large the signal is.

Interpretation:

- Smaller values mean the estimate is more precise.
- Larger values mean more uncertainty.

A useful mental model:

- High `log2_fc_estimate` with low `log2_fc_stderr` is more convincing.
- High `log2_fc_estimate` with high `log2_fc_stderr` is less certain.

The standard error depends on assay noise and the amount of information available for estimating the effect.

## The Hypothesis-Testing Columns

These columns quantify whether the observed effect is likely to be real rather than explained by assay noise.

### `t_statistic`

This is the test statistic for the estimated effect relative to the null hypothesis of no effect.

In simple terms, it compares signal size to its uncertainty:

`t_statistic ~= log2_fc_estimate / log2_fc_stderr`

Interpretation:

- Large positive values indicate strong evidence for activation.
- Values near `0` indicate weak evidence.
- Large negative values indicate evidence for reduced signal.

The bigger the magnitude of the t-statistic, the less compatible the data are with a no-effect assumption.

### `p_value`

This is the probability, under the null hypothesis of no true effect, of observing a result at least as extreme as the one measured.

Interpretation:

- Small `p_value` means the signal is unlikely to be due to random noise alone.
- Large `p_value` means the data are compatible with no real effect.

Important limitation:

- A p-value is not the probability that the compound is active.
- It does not measure effect size.
- With large screens, raw p-values alone are not enough because many compounds are tested at once.

### `fdr_bh`

This is the Benjamini-Hochberg false discovery rate adjusted p-value.

This is usually the more useful significance column in a screening context because thousands of tests are performed in parallel. Without correction, many apparently significant rows would arise by chance.

Interpretation:

- Smaller `fdr_bh` means stronger evidence after accounting for multiple testing.
- A threshold like `fdr_bh < 0.05` is commonly used to control the expected false discovery rate at 5%.

This makes `fdr_bh` more appropriate than `p_value` for hit calling across the whole screen.

### `neg_log10_fdr`

This is a transformed version of `fdr_bh`:

`neg_log10_fdr = -log10(fdr_bh)`

Why it exists:

- Very small FDR values are easier to compare on a log scale.
- Larger values mean stronger statistical significance.

Examples:

- `fdr_bh = 0.1` gives `neg_log10_fdr = 1`
- `fdr_bh = 0.01` gives `neg_log10_fdr = 2`
- `fdr_bh = 0.001` gives `neg_log10_fdr = 3`

This column is especially useful for plots such as volcano-style displays.

## Replication and Effect Size

### `n_replicates`

This is the number of replicate observations supporting the row-level summary.

Interpretation:

- Higher values generally provide more stable estimates.
- Lower values mean the summary is supported by less direct evidence.

In this dataset, many library rows are effectively singlicate, while control rows may have more replication.

### `cohens_d`

This is an effect size statistic. It measures how large the signal is relative to variability.

Unlike p-values, effect size tries to answer: "How big is the effect?" rather than only "Is the effect statistically detectable?"

Interpretation:

- Larger values indicate a stronger separation from baseline relative to noise.
- Smaller values indicate weaker practical effect, even if the p-value is small.

This is useful because a row can be statistically significant but still have a modest biological effect, or vice versa.

## How the Columns Work Together

No single column should be interpreted in isolation.

A practical interpretation pattern is:

1. Look at `log2_fc_estimate` to judge biological direction and magnitude.
2. Check `log2_fc_stderr` to see how precise that estimate is.
3. Use `fdr_bh` to judge statistical significance in the context of the whole screen.
4. Use `median_log2_fc` and `n_replicates` to assess robustness.
5. Use `cohens_d` to judge standardized effect size.

## Example Interpretations

### Strong and convincing hit

Suppose a row has:

- `log2_fc_estimate = 1.8`
- `log2_fc_stderr = 0.12`
- `t_statistic = 15`
- `fdr_bh = 1e-6`
- `median_log2_fc = 1.7`

This would indicate:

- A large positive signal.
- High precision.
- Very strong statistical support.
- Agreement between the estimate and the median summary.

This is the profile of a convincing active measurement at that concentration.

### Weak or noisy signal

Suppose a row has:

- `log2_fc_estimate = 0.3`
- `log2_fc_stderr = 0.25`
- `t_statistic = 1.2`
- `fdr_bh = 0.4`

This would indicate:

- A small positive effect.
- High uncertainty relative to the effect size.
- Weak statistical support after correction.

This would not be a convincing screening hit.

## Important Caveats

- This file contains single-concentration screening data, not full dose-response fits.
- A strong signal here does not by itself tell you potency.
- A statistically significant signal here does not by itself prove PXR specificity; the counter-assay is still needed.
- Plate effects, run effects, and sparse replication can influence interpretation, so `plate_id` and `experiment_name` remain important context.

## Practical Rule of Thumb

For initial screening interpretation:

- Use `log2_fc_estimate` for signal strength.
- Use `fdr_bh` for statistical confidence.
- Use `median_log2_fc`, `n_replicates`, and `cohens_d` as supporting evidence.

That combination gives a much better picture than relying on any one column alone.
