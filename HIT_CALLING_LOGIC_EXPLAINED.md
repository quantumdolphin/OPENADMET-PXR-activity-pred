# Hit-Calling Logic for the Single-Concentration PXR Screen

This note explains the reasoning behind the script [`identify_single_concentration_hits.py`](/wsl$/Ubuntu/home/avranga1008/pxr-challenge/activity/scripts/identify_single_concentration_hits.py).

The goal of the script is not to produce a perfect final truth label. Its purpose is to create a transparent, biologically reasonable, first-pass hit list from the single-concentration screening dataset:

- [`pxr-challenge_single_concentration_TRAIN.csv`](/wsl$/Ubuntu/home/avranga1008/pxr-challenge/activity/data/raw/pxr-challenge_single_concentration_TRAIN.csv)

The script is intentionally simple enough to inspect, modify, and defend.

## Background

### What this dataset represents

The single-concentration file is an early-stage screening table. Each row is one assay-level measurement of a compound at a specific concentration, plate, and experiment.

This is different from the downstream dose-response dataset:

- the single-concentration file tells us what happened at one tested concentration
- the dose-response file summarizes what happened across a fitted curve

So in this file, we are doing **screening hit identification**, not potency estimation.

### What a “hit” means in this context

A hit is a compound that appears to produce a meaningful signal in the assay and does so strongly enough that the signal is unlikely to be explained by assay noise alone.

That definition has two parts:

1. The signal should be statistically convincing.
2. The signal should be biologically large enough to matter.

Using only one of these is usually weak:

- a very small effect can be statistically significant in a large screen
- a large-looking effect can still be unreliable if it is noisy

That is why the script uses a hybrid rule.

## Why not use a single hard cutoff?

A pure threshold like:

- `log2_fc_estimate > 1`

is easy to understand, but it ignores statistical uncertainty.

A pure significance rule like:

- `fdr_bh < 0.05`

is also incomplete, because it can admit tiny effects that may not be useful biologically.

The script therefore combines:

- significance
- effect size
- robustness

This is conceptually similar to screening pipelines such as `tcpl`, where hit calling is not treated as a one-number decision.

## The columns used by the script

The script relies mainly on three columns from the single-concentration dataset.

### `log2_fc_estimate`

This is the main signal estimate.

It represents the log2 fold-change in reporter activity relative to baseline.

Interpretation:

- `0` means no change
- `1` means about a 2-fold increase
- `2` means about a 4-fold increase

This is the main biological effect-size column.

### `fdr_bh`

This is the Benjamini-Hochberg false discovery rate adjusted p-value.

It is used instead of raw `p_value` because thousands of rows are screened in parallel. The FDR correction helps control false positives in that multiple-testing setting.

This is the main statistical confidence column.

### `median_log2_fc`

This is a more robust signal summary.

The script uses it as a guardrail against rows where the fitted estimate looks strong but the central tendency of the measurement is less convincing.

This is the main robustness column.

## The core hit-calling idea

The script makes decisions in two stages:

1. Row-level hit calling
2. Compound-level aggregation across concentrations

That separation is important.

First we ask:

- is this particular measurement active at this particular concentration?

Then we ask:

- does the compound look convincing across the tested concentration range?

## Stage 1: Row-level hit calling

Each row is evaluated with three conditions.

### 1. Significance rule

```text
fdr_bh < 0.05
```

This means the row must survive multiple-testing correction.

The idea is:

- if the adjusted significance is weak, we should not trust the row as a hit no matter how visually large it looks

### 2. Effect-size rule

```text
log2_fc_estimate >= 1.0
```

This means the estimated signal should be at least about a 2-fold increase over baseline.

The idea is:

- the signal should not just be statistically detectable
- it should also be large enough to matter biologically

### 3. Robustness rule

```text
median_log2_fc >= 0.8
```

This means the median signal should also be elevated.

The exact threshold is slightly looser than the main effect-size threshold on purpose. It allows small differences between the fitted estimate and the median while still requiring broad agreement.

The idea is:

- avoid calling hits based only on one optimistic estimate
- require the row to look reasonably strong by a robust summary too

### Row-level hit definition

A row is called a hit only if all three conditions are true:

```text
row_hit =
    (fdr_bh < 0.05)
    AND (log2_fc_estimate >= 1.0)
    AND (median_log2_fc >= 0.8)
```

This is the main screening rule in the script.

## Why concentrations are handled separately first

The dataset contains multiple screening concentrations. A compound can be active at one concentration and inactive at another.

That means we should not collapse all concentrations immediately into one decision.

Instead, the script first asks:

- at which concentrations does this compound pass the row-level rule?

Only after that does it build a compound-level summary.

This is important because:

- lower-concentration activity is usually more interesting than activity only at the highest concentration
- repeated activity across adjacent concentrations is more convincing than a single isolated spike

## Stage 2: Compound-level aggregation

After row hits are identified, the script groups active rows by compound.

Because `Molecule Name` is often blank in this dataset, the script uses a fallback identifier strategy:

1. `Molecule Name`
2. `OCNT_ID`
3. `SMILES`

This is just a practical fix for the structure of the file.

For each compound, the script records:

- how many active rows it has
- how many unique active concentrations it has
- the list of active concentrations
- the lowest active concentration
- the highest active concentration
- the strongest observed signal
- the best FDR value

These summaries are then used for prioritization.

## Dose consistency

The script also checks whether the signal behaves in a roughly dose-consistent way across active concentrations.

This is not enforced as a strict mechanistic truth. It is a ranking heuristic.

The script allows a small decrease between adjacent active concentrations using a tolerance. By default, a later concentration can dip by up to 15% relative to the previous active one and still count as “dose consistent.”

The reason for this tolerance is practical:

- real screening data are noisy
- a nearly flat or slightly wobbly signal should not be penalized too aggressively

So the script is looking for approximate monotonic support, not perfect monotonicity.

## Tier assignment logic

After compound-level summaries are built, the script assigns each compound to one of three tiers.

### Tier 1

A compound becomes `Tier 1` if:

- it is active at a low or mid concentration (`<= 33 uM`)
- it has support at at least two active concentrations
- its active signals are dose consistent

Interpretation:

- these are the most attractive first-pass hits
- they do not rely only on the highest concentration
- they show some concentration-range support

### Tier 2

A compound becomes `Tier 2` if it does not qualify for `Tier 1`, but also is not only a highest-concentration-only hit.

Interpretation:

- these compounds still look interesting
- they may be active, but the evidence is weaker or less clean than Tier 1

### Tier 3

A compound becomes `Tier 3` if it is active only at the highest concentration.

Interpretation:

- these are the most cautious hits
- they may still be real
- but they carry more risk of nonspecific or concentration-driven artifacts

## Why this is a reasonable first-pass strategy

This logic is useful because it reflects the actual structure of screening data:

- row-level evidence matters
- concentration context matters
- not all hits should be treated equally

It also avoids two common mistakes:

### Mistake 1: using only a hard signal cutoff

That ignores uncertainty.

### Mistake 2: using only significance

That ignores biological relevance.

The script instead uses a hybrid framework that is:

- transparent
- easy to tune
- easy to inspect
- more defensible than a one-number rule

## What this script does not do

This script is deliberately a first-pass screen. It does not do the following:

- fit dose-response curves
- estimate potency such as `EC50` or `pEC50`
- use the counter-assay to remove nonspecific hits
- calibrate thresholds against downstream labels automatically

Those belong to later stages of analysis.

## How to interpret the outputs

The script writes four output tables:

- row-level hits
- compound-level hit summaries
- concentration-level row-hit summary
- tier summary

The most important table for prioritization is the compound-level table:

- [`single_concentration_compound_hits.csv`](/wsl$/Ubuntu/home/avranga1008/pxr-challenge/activity/outputs/single_concentration_hits/single_concentration_compound_hits.csv)

That table is the practical bridge from screening measurements to candidate selection.

## How to tune the logic

The default thresholds are not sacred. They are chosen to be understandable and biologically plausible.

The main tuning knobs are:

- `--fdr-threshold`
- `--effect-threshold`
- `--median-threshold`
- `--concentration-tolerance`

Examples of how tuning changes behavior:

- lower `fdr-threshold` gives fewer, stricter hits
- higher `effect-threshold` favors stronger activators
- higher `median-threshold` demands more robust support
- lower `concentration-tolerance` makes dose consistency stricter

## Recommended next step

The best next step is to compare these first-pass screening hits against the downstream dose-response training file:

- [`pxr-challenge_TRAIN.csv`](/wsl$/Ubuntu/home/avranga1008/pxr-challenge/activity/data/raw/pxr-challenge_TRAIN.csv)

That would let us answer:

- which thresholds best recover compounds that later show convincing dose-response behavior?

That comparison is the right way to move from a reasonable heuristic to a calibrated screening rule.
