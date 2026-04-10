
# Target Quality Interpretation

From a cheminformatics and assay-modeling perspective, these target-quality results are more important than any early model choice, because they define what kind of regression problem this really is. The main conclusion is that the PXR training labels do not behave like a clean, homoscedastic medicinal chemistry potency series. They behave like a mixed-quality assay readout with a reasonably well-behaved mid/high-potency regime and a distinctly noisier weak-activity regime. That distinction should shape how we train, validate, and interpret every later model.
![Alt text](pEC50_histogram.png)
The first point is the shape of the endpoint itself. The histogram in shows a broad, asymmetric potency distribution with most compounds concentrated around roughly `4.5-5.4`, but with a long left tail extending down to `1.61`. In a classical SAR series, one often hopes for a smoother unimodal potency spread around a chemically coherent range. Here, the low-potency side looks more like a mixture of weak true actives, borderline measurements, and compounds approaching assay detection limits. That interpretation is strengthened by the finer binning in , where the region below about `2.5-3.0` does not read as a smooth continuation of the main potency population. As a chemist, I would treat that lower region as a lower-confidence response regime rather than just “weaker compounds.”

![Alt text](pEC50_fine_histogram.png)

That matters because `pEC50` is already a log-transformed endpoint, so the usual first instinct, “should the target be transformed?”, is probably the wrong question. The more relevant question is whether all parts of the target range should be trusted equally. The answer appears to be no. The absence of strong rounding artifacts is actually helpful here. The summary table [summary_metrics.csv](/home/avranga1008/pxr-challenge/activity/outputs/eda/target_quality/tables/summary_metrics.csv) reports only about `1.0%` integer-valued and `1.7%` half-step-valued `pEC50`s, so the low-end irregularity is not just a formatting or curation artifact. It looks more like a genuine assay-quality phenomenon.

![Alt text](stderr_vs_pEC50.png)

The strongest evidence comes from the uncertainty plots. In  `pEC50_std.error` decreases sharply as potency increases. The summary gives a Spearman correlation of `-0.850`, which is extremely strong for this kind of assay-quality relationship. Practically, that means the weak end is not just lower in activity; it is much less precisely measured. The reported median standard error drops from about `0.57` in the weakest decile to about `0.057` in the strongest decile. From a QSAR standpoint, that is heteroscedastic noise of a magnitude large enough to distort model fitting if all rows are weighted equally.
![Alt text](ci_width_vs_pEC50.png)
The confidence intervals tell the same story and make the case even stronger. In CI width collapses as potency increases, with median width falling from about `2.02` in the weakest decile to about `0.24` in the strongest. The corresponding Spearman correlation in  is `-0.848`. In medicinal chemistry terms, this means the weak region is not just biologically weaker; it is experimentally blurrier. For modeling, that is the clearest justification for uncertainty-aware training or at least sample-weighted regression.

That is why the statement “weak compounds are systematically noisier” is not just a descriptive observation. It is the core statistical property of this endpoint. It argues against treating low-end `pEC50` values as equally trustworthy anchors for SAR learning. A model trained naively on all rows with equal weight will spend substantial capacity trying to explain what is partly measurement variance. The more chemically sensible position is to preserve those rows, because they still contain information, but to downweight them relative to the tighter, better-defined mid/high-potency region. The file is therefore best interpreted as a weighting/QC aid, not as an exclusion list. For a first-pass model, cautious downweighting is more defensible than aggressive pruning.

It is also useful that the heuristic search for “high uncertainty but apparently strong potency” did not surface obvious pathological compounds. The file is empty. That suggests the dataset is not dominated by bizarre, high-potency curve-fit failures. The uncertainty problem is more systematic and monotonic: weaker compounds are less reliable overall.

![Alt text](pEC50_vs_Emax_colored_by_stderr.png)

The `Emax` relationship is also chemically informative. In potency and efficacy are only weakly coupled, and the reported Spearman correlation is `-0.140`. That is weak enough that `Emax` should not be treated as a surrogate for `pEC50`. From a receptor pharmacology perspective, this is not surprising. Potency and maximal effect often encode different aspects of compound behavior, especially in cell-based transcriptional reporter systems. A compound can look reasonably potent yet have modest efficacy, or conversely show broad transcriptional response without especially clean potency. So `Emax` is better treated as an auxiliary mechanistic or QC signal, not as an interchangeable label with potency.

The modeling implication is straightforward. This target should be treated as continuous regression, but not as simple equal-confidence regression. The chemistry is not telling us to bin the target yet; the assay is telling us to respect uncertainty. For that reason, the most defensible baseline strategy is:

- keep raw `pEC50` as the target
- avoid unnecessary target transforms
- use uncertainty-derived sample weights rather than changing the endpoint
- evaluate models with validation schemes that do not leak close analogs

Overall, the target-quality EDA supports a view of this dataset as a heteroscedastic potency prediction problem with a chemically usable high-confidence regime and a much noisier weak-activity regime. That should remain central when choosing representations, tuning models, and interpreting leaderboard performance.
