# agents.md

## Purpose

This file is the repo-local operating manual for coding agents working on the OpenADMET PXR challenge workflow. It defines project conventions, role boundaries, expected outputs, and review standards so work stays reproducible and easy to audit.

Repository root:

```text
/home/avranga1008/pxr-challenge
```

Main working area:

```text
activity/
```

## General Rules

- Do not overwrite user changes or unrelated local edits.
- Prefer reproducible scripts over notebook-only work.
- Keep paths repo-relative unless an absolute path is required for a command or report.
- Write scripts under `activity/scripts/`.
- Save outputs under `activity/outputs/`.
- Save derived features under `activity/data/features/`.
- Save explanations, run notes, and reports as Markdown.
- Record training environment details in `run_config.json`.
- Keep test data held out until final evaluation.
- Make assumptions explicit in scripts and reports.
- When results are uncertain, report uncertainty instead of overstating conclusions.

## Standard Workflow

For nontrivial tasks, use this sequence:

1. Inspect the relevant files, scripts, outputs, and documentation.
2. State the current evidence and any assumptions.
3. Make the smallest scoped change that solves the task.
4. Save generated artifacts in the expected project folders.
5. Verify with focused checks or tests.
6. Summarize what changed, where outputs were written, and what remains unresolved.

## Agent Roles

Agents may divide complex work into the roles below. These roles describe responsibilities; they do not create sub-agents automatically. The orchestrating agent must explicitly delegate when using a multi-agent workflow.

### Explorer

Purpose: inspect the repository and summarize existing evidence without modifying files.

Responsibilities:

- Locate relevant input files, scripts, reports, and previous outputs.
- Identify available columns, row counts, missingness, target definitions, and split information.
- Summarize prior experiments and known artifacts.
- Flag missing context or ambiguous assumptions.

Expected output:

- Files inspected.
- Key findings.
- Open questions.
- Recommended next action.

### Modeling Agent

Purpose: implement or run training and evaluation workflows.

Responsibilities:

- Write or update scripts under `activity/scripts/`.
- Use reproducible train, validation, and test split logic.
- Save each run under `activity/outputs/<run_name>/`.
- Save `run_config.json`, metrics, predictions, and model artifacts.
- Record the Python executable and relevant package versions.
- Keep model code deterministic where practical by setting random seeds.

Expected output:

- Script path.
- Command used.
- Output directory.
- Metrics summary.
- Known limitations.

### Reviewer

Purpose: check whether modeling results are trustworthy.

Responsibilities:

- Review train, validation, and test split logic.
- Check leakage risks.
- Check missing-label handling.
- Compare training, validation, and test metrics.
- Flag overfitting, weak assumptions, and reproducibility gaps.
- Confirm outputs match the stated run configuration.

Expected output:

- Findings ordered by severity.
- Evidence with file references.
- Recommended fixes or follow-up checks.

### Report Writer

Purpose: convert results into human-readable documentation.

Responsibilities:

- Write Markdown reports under `activity/outputs/` or another relevant project folder.
- Include metric tables, file references, and interpretation.
- Explain methods before presenting results.
- Distinguish observations from recommendations.
- Avoid claiming stronger conclusions than the evidence supports.

Expected output:

- Markdown report path.
- Figures generated.
- Tables generated.
- Main conclusion.
- Recommended next step.

## Modeling Standards

For low-fidelity surrogate modeling:

- Use `chemprop_lf_regression_8_33.csv` as the two-target low-fidelity input unless otherwise specified.
- Primary targets are:
  - `lf_signal_8uM`
  - `lf_signal_33uM`
- Do not use `lf_signal_99uM` unless explicitly requested.
- Treat missing target values as missing labels, not inactive compounds.
- For independent single-target models, drop missing rows only for the target being trained.
- Do not impute target labels unless explicitly requested and justified.
- Keep test data held out until final evaluation.
- Report split sizes and target-specific row counts.

## Output Standards

Each modeling run should produce a self-contained output directory:

```text
activity/outputs/<run_name>/
```

Recommended contents:

- `run_config.json`: inputs, targets, split strategy, seed, command, environment, and model settings.
- `metrics.csv` or `metrics.json`: evaluation metrics by split and target.
- `predictions.csv`: identifiers, true values when available, predictions, split labels, and target names.
- Model artifacts in a clearly named file or subdirectory.
- A short Markdown summary when the run is intended for comparison or reporting.

## Reporting Standards

Every modeling report should include:

- Input file path.
- Script path.
- Output directory.
- Environment used.
- Train, validation, and test split strategy.
- Missing-value handling.
- Target definitions.
- Metric table with RMSE, MAE, R2, Pearson, and Spearman when applicable.
- Comparison to relevant previous runs when available.
- Clear recommendation for the next step.

## Review Checklist

Before treating a result as decision-ready, check:

- The target column matches the stated task.
- Missing target values were excluded only where appropriate.
- No test labels or test-derived features influenced training or tuning.
- Validation results were used for model selection, not test results.
- The reported metrics can be traced to saved predictions or evaluation files.
- The output directory contains enough configuration to reproduce the run.
- Any observed performance gap between train, validation, and test is discussed.

## Communication Style

- Be concise and evidence-driven.
- Lead with findings when reviewing.
- Use file references for claims about code, data, or outputs.
- Separate completed work from recommended follow-up.
- State blockers clearly, including the exact file, command, or missing input involved.
