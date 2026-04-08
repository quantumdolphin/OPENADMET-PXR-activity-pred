# Understanding the PXR Training Dataset

Before working with this dataset, it helps to understand what kind of experiment generated it. These data come from a cell-based assay designed to measure activation of the human pregnane X receptor (PXR). When a compound activates PXR, it triggers expression of a reporter gene (luciferase), and that signal is what ultimately gets measured. The dataset is therefore not directly measuring binding, but a downstream biological response that reflects receptor activation.

The training data are divided into three files, each representing a different level of experimental detail. The main file contains dose–response summaries, the counter-assay file checks specificity, and the single-concentration file contains raw screening-level measurements.

---

## The Main Dataset: Dose–Response Summary

The file `pxr-challenge_TRAIN.csv` is the most important one. Each row represents a single molecule, and the values in that row come from fitting a dose–response curve to experimental data.

The first few columns describe the identity of the molecule.

“Molecule Name” is the most reliable identifier for a compound and is the key used to connect entries across all files. For example, a name like "ENAMINE\_ABC123" refers to the same chemical entity wherever it appears. In the dose–response file it appears once with summary values, in the counter-assay it appears again if tested for specificity, and in the single-concentration file it may appear multiple times because the same molecule is measured at different concentrations or plates. Despite these multiple rows, the name always refers to the same underlying compound.

“OCNT\_ID” is different. It does not identify the molecule itself, but a specific tested sample or measurement instance within the single-concentration dataset. The same OCNT\_ID can appear across multiple rows because that sample is measured at different concentrations or on different plates. It is therefore useful for tracking experimental measurements, but it cannot be used to connect the different CSV files. For cross-file linking, you should always rely on “Molecule Name”.

“SMILES” is a string representation of the chemical structure. This is what you will use for cheminformatics or machine learning, because it encodes the atoms and bonds of the molecule. “OCNT Batch” is a batch identifier that tracks how the compound was handled experimentally, but it should be treated as metadata rather than a primary identifier.

Potency

The next group of columns describes potency. The key value here is pEC50. This is defined as the negative logarithm of the EC50 value, where EC50 is the concentration of compound required to achieve half of the maximal response. In this assay, “maximal response” is defined in terms of the luciferase signal expressed as log2 fold change (log2FC) relative to baseline. Imagine the baseline signal is 1 (no activation). If a compound reaches an Emax of log2FC = 2, that corresponds to a 4-fold increase in signal (since 2^2 = 4). Half of this maximal response is not log2FC = 1 in a linear sense, but rather half of the *fold-change effect*. Half of 4-fold is 2-fold, which corresponds to log2FC = 1. So the EC50 is the concentration at which the observed signal reaches a log2FC of \~1 in this example. This is why EC50 depends on the full dose–response curve: it is defined relative to the compound’s own maximum effect, not an absolute signal value. The use of the log2 scale compresses large fold-changes and makes the response approximately linear with respect to underlying transcriptional activity, which helps with modeling and comparison across compounds. In this assay, the “response” is the luminescence from a luciferase reporter driven by PXR activation. It is not an absolute concentration or binding signal; instead, it is normalized relative to untreated cells and reported as a fold change. Practically, this dataset expresses response as log2 fold change (log2FC), which is dimensionless. Therefore, EC50 is in units of molarity (M), while pEC50 is unitless (because it is −log10 of molarity), and Emax/log2FC values represent relative signal intensity rather than physical units like concentration or energy.

It is useful to think about an effective “ratio-like” interpretation even though no formal ratio is defined. For example, compare two compounds: one with pEC50 = 7 and Emax = 1, and another with pEC50 = 6 and Emax = 2. The first works at 100 nM but only doubles the signal (2×), while the second needs 1 µM but produces a 4× signal. In practice, you are balancing how little drug you need (potency) against how strong the response is (efficacy). Informally, a compound with high pEC50 and high Emax has a favorable profile because it achieves a large effect at low concentration. This “balance” is what people sometimes intuitively treat like a ratio: it captures efficiency of producing biological response per unit concentration, even though mathematically pEC50 and log2FC come from different parts of the dose–response curve. Because of the negative log scale, higher pEC50 values correspond to more potent compounds. For example, a pEC50 of 6 corresponds to a micromolar compound, while a pEC50 of 7 corresponds to a sub-micromolar compound. The dataset also includes lower and upper confidence bounds for pEC50, as well as a standard error. These values tell you how reliable the fitted potency is. If the confidence interval is wide, it usually means the experimental curve was noisy or poorly defined.

Efficacy

The next set of columns describes efficacy, which is how strong the response is once the compound activates the receptor. This is captured by “Emax\_estimate (log2FC vs. baseline)”. This value is expressed as a log2 fold change relative to baseline signal. A value of 1 means the signal doubled, while a value of 2 means a four-fold increase. This is important because two compounds can have the same potency but very different maximal effects. As with pEC50, there are confidence intervals and standard errors for Emax that reflect uncertainty in the estimate.

There is also a set of columns labeled “Emax.vs.pos.ctrl”. These values normalize the maximal response relative to a positive control compound included in the assay. A value of 1 means the compound is as effective as the control, while values below or above 1 indicate weaker or stronger responses. This normalization helps reduce variability between experimental runs.

Finally, the “Split” column indicates how the dataset is partitioned for modeling, such as training or validation. This is mainly relevant when building predictive models.

---

## Interpreting pEC50 and log2FC with Concrete Examples

It helps to see real numbers to understand what these values mean in practice.

Consider a compound with pEC50 = 6 and Emax (log2FC) = 2. A pEC50 of 6 corresponds to an EC50 of 10⁻⁶ M, or 1 µM. This means you need about 1 micromolar concentration of the compound to reach half of its maximal effect. The Emax value of 2 (in log2 fold change units) corresponds to a 4-fold increase in signal relative to baseline, since 2² = 4. So at high concentration, this compound drives about a four-fold increase in luciferase signal.

Now compare this to another compound with pEC50 = 7 and Emax (log2FC) = 1. A pEC50 of 7 corresponds to an EC50 of 10⁻⁷ M, or 100 nM, meaning it is more potent and works at lower concentration. However, its Emax of 1 corresponds to only a 2-fold increase in signal. So this compound is more potent but produces a weaker maximal response.

These two values capture different aspects of pharmacology. pEC50 tells you how much compound is needed to activate the receptor, while log2FC (Emax) tells you how strong the biological response is once the receptor is activated. There is no formal ratio between these values, but together they define the overall behavior of a compound.

This is why both are needed. A compound can be very potent but weak in effect, or very strong in effect but require high concentrations. In drug discovery, the goal is usually to find compounds that combine high potency (high pEC50) with strong efficacy (high Emax), while also being specific based on the counter-assay.

---

## The Counter-Assay Dataset: Checking Specificity

The file `pxr-challenge_counter-assay_TRAIN.csv` has the same columns as the main dataset, but the biological meaning is different. This assay is performed in a modified cell line that does not have functional PXR. The rest of the experimental system is the same, including the reporter gene.

The purpose of this assay is to detect non-specific effects. If a compound appears active in the main assay but also shows activity in the counter-assay, then its signal is likely not due to PXR activation. Instead, it may be affecting the reporter system directly, altering transcription in a general way, or causing toxicity-related artifacts.

This means you should always interpret the main dataset together with the counter-assay. A compound that is active in the main assay but inactive in the counter-assay is a strong candidate for true PXR activation. A compound that is active in both is likely a false positive.

---

## The Single-Concentration Dataset: Raw Screening Data

The file `pxr-challenge_single_concentration_TRAIN.csv` represents an earlier stage of the experimental pipeline. Instead of fitting a full dose–response curve, each compound is tested at one or a few concentrations, often across multiple plates and replicates.

Because of this, each molecule appears multiple times in this file. The “OCNT\_ID” column refers to a specific measurement instance rather than a unique molecule. For example, imagine a molecule called "Mol\_A". In this file, you might see rows like:

- OCNT\_ID = OCNT\_10234, concentration = 10 µM, plate = P1
- OCNT\_ID = OCNT\_10234, concentration = 30 µM, plate = P1
- OCNT\_ID = OCNT\_10234, concentration = 10 µM, plate = P2

Here, OCNT\_10234 represents the same tested sample, but measured multiple times across different concentrations and plates. This is why OCNT\_ID repeats across rows.

Importantly, OCNT\_ID does not connect the different CSV files. It only exists in the single-concentration dataset and is tied to individual experimental measurements, not to the canonical molecule identity. To connect across files, you should use “Molecule Name”, which represents the actual compound.

“plate\_id” identifies the experimental plate, and “experiment\_name” identifies the experimental run. These are important for understanding batch effects and experimental variability.

The “concentration\_M” column gives the concentration at which the compound was tested. The main measurement is “log2\_fc\_estimate”, which is the log2 fold change in signal relative to baseline. This is similar in concept to Emax, but measured at a single concentration rather than across a full curve.

Several statistical columns help assess whether the observed signal is meaningful. The “t\_statistic”&#x20;

&#x20;many compounds are tested simultaneously, the dataset also includes “fdr\_bh”, which is a multiple-testing corrected p-value using the Benjamini–Hochberg method. The “neg\_log10\_fdr” is simply a transformed version that makes strong signals easier to identify.

Additional columns such as “median\_log2\_fc”, “n\_replicates”, and “cohens\_d” summarize the consistency and effect size of the signal across replicates. These help distinguish real biological activity from noisy measurements.

---

## How the Three Files Fit Together

You can think of these three datasets as representing different stages of a drug discovery screening workflow.

The single-concentration dataset is the first pass. It is fast and high-throughput, allowing thousands of compounds to be screened quickly, but it provides only limited information.

The dose–response dataset is the second stage. Here, selected compounds are tested more carefully across multiple concentrations, allowing accurate estimation of potency (pEC50) and efficacy (Emax).

The counter-assay is the validation step. It filters out compounds that produce signals for reasons unrelated to PXR, ensuring that the final dataset reflects true biological activity.

---

## Key Takeaways

The most important idea is that each column connects back to a biological or statistical concept. pEC50 tells you how much compound is needed to activate the receptor. Emax tells you how strong that activation is. Confidence intervals and standard errors tell you how reliable those values are. The counter-assay tells you whether the activity is truly due to PXR. The single-concentration data provide the raw signals from which these summaries are derived.

Once you understand these concepts, the dataset becomes much easier to interpret and use for modeling or mechanistic analysis.

