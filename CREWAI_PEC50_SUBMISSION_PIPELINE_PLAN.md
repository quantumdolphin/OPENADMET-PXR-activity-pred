# CrewAI pEC50 Submission Pipeline Plan

## Purpose

This document describes a proposed CrewAI-based orchestration workflow for building a submission-ready `pEC50` prediction pipeline for the OpenADMET PXR challenge.

The goal is to automate the repeatable parts of the modeling process:

- Validate input data.
- Build molecular feature tables.
- Train multiple machine learning models.
- Compare models on reproducible validation/holdout splits.
- Select a champion model or ensemble.
- Generate blinded test predictions.
- Write a challenge-ready submission CSV.
- Produce review reports and method documentation.

CrewAI should be used as the orchestration layer. The actual modeling logic should remain in normal Python scripts so the workflow stays reproducible, inspectable, and easy to rerun without agent infrastructure.

## High-Level Pipeline

Input files:

```text
activity/data/raw/pxr-challenge_TRAIN.csv
activity/data/raw/pxr-challenge_TEST_BLINDED.csv
activity/data/features/single_dose_chemprop/chemprop_lf_regression_8_33.csv
```

Main outputs:

```text
activity/outputs/submissions/<run_name>/submission.csv
activity/outputs/submissions/<run_name>/submission_predictions_detailed.csv
activity/outputs/submissions/<run_name>/run_config.json
activity/outputs/submissions/<run_name>/method_report.md
```

The final submission CSV should contain the blinded test molecule identifiers and predicted `pEC50` values in the challenge-required format.

## Recommended Repository Structure

```text
activity/
  crewai_pipeline/
    crew.py
    agents.py
    tasks.py
    tools.py
    config/
      agents.yaml
      tasks.yaml
    workflows/
      pec50_submission_pipeline.yaml

  scripts/
    validate_pxr_inputs.py
    build_pec50_features.py
    train_pec50_ridge.py
    train_pec50_random_forest.py
    train_pec50_hgb.py
    train_pec50_xgboost.py
    train_pec50_chemprop.py
    train_pec50_stacking_ensemble.py
    predict_blinded_test.py
    make_submission_csv.py
    write_submission_report.py

  outputs/
    crew_runs/
    model_runs/
    submissions/
```

The scripts should be runnable directly from the command line. CrewAI should call those scripts, collect their artifacts, and make workflow decisions from structured outputs such as `metrics.json`, `run_config.json`, and leaderboard CSV files.

## Agent Roles

### 1. Data QA Agent

Purpose: verify that the raw challenge data is valid and ready for modeling.

Responsibilities:

- Check required columns.
- Check duplicate molecule identifiers.
- Check duplicate SMILES.
- Check missing `pEC50` labels in training data.
- Check SMILES validity.
- Check train/test schema consistency.
- Check whether uncertainty/error columns are available.
- Write a data QA report.

Expected outputs:

```text
activity/outputs/crew_runs/<run_name>/data_qa_report.md
activity/outputs/crew_runs/<run_name>/data_qa_summary.json
```

### 2. Feature Engineering Agent

Purpose: build model-ready train and blinded-test feature matrices.

Responsibilities:

- Generate RDKit descriptors.
- Generate Morgan fingerprints.
- Generate charge features if available.
- Merge predicted low-fidelity surrogate features.
- Create aligned train/test modeling matrices.
- Save feature summaries.
- Record feature-generation parameters.

Candidate feature groups:

```text
desc_only
desc_plus_fp
desc_plus_charge_fp
desc_plus_lf
desc_plus_fp_plus_lf
desc_plus_charge_fp_plus_lf
```

Expected outputs:

```text
activity/data/features/<run_name>/train_features.csv
activity/data/features/<run_name>/test_features.csv
activity/data/features/<run_name>/feature_summary.json
```

### 3. Modeling Agent

Purpose: train candidate `pEC50` prediction models.

Candidate model families:

- Ridge / ElasticNet baseline.
- Random Forest.
- HistGradientBoosting.
- XGBoost.
- LightGBM, if installed.
- Chemprop.
- Chemprop plus auxiliary low-fidelity features, if practical.
- Stacking or blending ensemble.

Responsibilities:

- Use fixed train/validation/holdout or cross-validation splits.
- Train candidate models.
- Save validation/holdout predictions.
- Save model artifacts.
- Save environment metadata.
- Save a `run_config.json` for each model run.

Expected outputs:

```text
activity/outputs/model_runs/<run_name>/<model_name>/model.*
activity/outputs/model_runs/<run_name>/<model_name>/metrics.json
activity/outputs/model_runs/<run_name>/<model_name>/holdout_predictions.csv
activity/outputs/model_runs/<run_name>/<model_name>/run_config.json
```

### 4. Evaluation Agent

Purpose: rank models and determine whether the apparent winner is trustworthy.

Responsibilities:

- Compare RMSE, MAE, R2, Pearson, and Spearman.
- Check train/validation/holdout gaps.
- Check residual bias.
- Check performance by `pEC50` range.
- Check high-activity compound performance.
- Check uncertainty-weighted metrics if assay error columns are available.
- Flag likely overfitting.
- Recommend a champion model or ensemble.

Expected outputs:

```text
activity/outputs/model_runs/<run_name>/model_leaderboard.csv
activity/outputs/model_runs/<run_name>/evaluation_report.md
```

### 5. Submission Agent

Purpose: refit the selected champion model and generate blinded test predictions.

Responsibilities:

- Refit the selected model on all allowed labeled training rows.
- Predict the blinded test rows.
- Clip or sanity-check predicted `pEC50` values if needed.
- Write the final challenge-format submission CSV.
- Save detailed prediction diagnostics.
- Save final model configuration.

Expected outputs:

```text
activity/outputs/submissions/<run_name>/submission.csv
activity/outputs/submissions/<run_name>/submission_predictions_detailed.csv
activity/outputs/submissions/<run_name>/run_config.json
```

### 6. Review Agent

Purpose: act as a safety gate before a file is treated as submission-ready.

Responsibilities:

- Verify no train/test leakage.
- Verify blinded test order and molecule identifiers.
- Verify row count equals the challenge blinded test row count.
- Verify no missing predictions.
- Verify no infinite predictions.
- Verify expected output column names.
- Check that final model selection did not use blinded test labels.
- Check that all scripts, configs, and artifacts were saved.

Expected outputs:

```text
activity/outputs/submissions/<run_name>/submission_review.md
activity/outputs/submissions/<run_name>/submission_checks.json
```

### 7. Report Agent

Purpose: write the final method report.

Responsibilities:

- Summarize the data.
- Summarize feature generation.
- Summarize models tried.
- Explain champion selection.
- Include leaderboard tables and plots.
- List final submission files.
- Explain limitations and risks.

Expected output:

```text
activity/outputs/submissions/<run_name>/method_report.md
```

## CrewAI Flow

Recommended first implementation: sequential workflow.

```text
Data QA
  -> Feature Engineering
  -> Candidate Model Training
  -> Evaluation
  -> Champion Selection
  -> Final Refit
  -> Blinded Test Prediction
  -> Submission Validation
  -> Method Report
```

CrewAI conceptual structure:

```python
from crewai import Agent, Crew, Process, Task

crew = Crew(
    agents=[
        data_qa_agent,
        feature_agent,
        modeling_agent,
        evaluation_agent,
        submission_agent,
        review_agent,
        report_agent,
    ],
    tasks=[
        data_qa_task,
        feature_task,
        model_training_task,
        evaluation_task,
        submission_task,
        review_task,
        report_task,
    ],
    process=Process.sequential,
)
```

Do not start with a fully autonomous parallel modeling workflow. First build a stable sequential process. Parallel model training can be added later after the scripts and artifact contracts are stable.

## Model Strategy

Use a tiered modeling ladder.

### Tier 1: Sanity Baselines

- Ridge.
- ElasticNet.
- Random Forest.
- HistGradientBoosting.

Purpose: establish fast baseline performance and catch data leakage or feature bugs.

### Tier 2: Strong Tabular Models

- XGBoost with descriptors.
- XGBoost with descriptors and Morgan fingerprints.
- XGBoost with descriptors, fingerprints, and predicted low-fidelity features.

Purpose: produce strong interpretable tabular models and feature importances.

### Tier 3: Graph Neural Models

- Chemprop `pEC50` model.
- Chemprop variants with auxiliary low-fidelity predictions, if practical.

Purpose: use molecular graph structure directly rather than relying only on hand-engineered descriptors and fingerprints.

### Tier 4: Ensemble Models

- Weighted average of top models.
- Stacking model trained on out-of-fold predictions.
- Simple blend of Chemprop plus XGBoost if they make complementary errors.

Purpose: reduce model-specific error if candidate models have partially independent failure modes.

## Low-Fidelity Feature Integration

The low-fidelity surrogate models can provide auxiliary features for downstream `pEC50` prediction.

Primary LF surrogate features:

```text
predicted_screen_signal_8uM
predicted_screen_signal_33uM
```

Optional XGBoost control features:

```text
xgb_predicted_screen_signal_8uM
xgb_predicted_screen_signal_33uM
```

Feature sets to compare:

```text
base_descriptors
base_descriptors_fingerprints
base_descriptors_fingerprints_lf_chemprop
base_descriptors_fingerprints_lf_chemprop_xgb
```

The Evaluation Agent should explicitly answer:

- Do predicted LF features improve `pEC50` holdout RMSE?
- Do predicted LF features improve ranking of active compounds?
- Do predicted LF features cause leakage risk?
- Are LF feature gains stable across seeds?

Important rule: predicted LF features should be generated from models that do not use `pEC50` labels. They are acceptable auxiliary molecular features only if their training process remains independent of the downstream `pEC50` target.

## Metrics

Every model should report:

```text
train RMSE
validation RMSE
holdout RMSE
holdout MAE
holdout R2
holdout Pearson
holdout Spearman
train-holdout gap
```

When assay uncertainty columns are available, also report uncertainty-aware metrics or weighted training summaries.

The champion should not be selected on one number alone. The Evaluation Agent should consider:

- Primary metric: holdout RMSE.
- Secondary metrics: R2, Pearson, Spearman.
- Overfitting: train/holdout gap.
- Stability: seed-to-seed variability.
- Submission risk: prediction range, missing outputs, reproducibility.

## Submission Safety Checks

Before writing or approving a final submission CSV, the Review Agent must verify:

- Number of prediction rows equals blinded test rows.
- Molecule IDs match the blinded test file exactly.
- No missing predictions.
- No infinite predictions.
- No duplicate molecule IDs unless duplicates exist in the input.
- Prediction column name matches the challenge requirement.
- Predicted `pEC50` values are finite.
- Predicted `pEC50` range is chemically plausible.
- Final model was trained only on allowed training labels.
- No blinded test labels were used.
- All scripts, configs, and model artifacts are saved.

## Example Workflow Configuration

Create one YAML config per experiment.

```yaml
run_name: pec50_crew_run_001
seed: 42

paths:
  train_csv: activity/data/raw/pxr-challenge_TRAIN.csv
  test_csv: activity/data/raw/pxr-challenge_TEST_BLINDED.csv
  output_dir: activity/outputs/crew_runs/pec50_crew_run_001

splits:
  method: random
  train: 0.8
  val: 0.1
  holdout: 0.1

features:
  rdkit_descriptors: true
  morgan_fingerprints: true
  morgan_radius: 2
  morgan_bits: 2048
  predicted_lf_features: true

models:
  ridge: true
  random_forest: true
  hgb: true
  xgboost: true
  chemprop: false
  ensemble: true

selection:
  primary_metric: holdout_rmse
  secondary_metrics:
    - holdout_r2
    - spearman
    - train_holdout_gap
```

## Tools For CrewAI

Agents should not directly invent modeling logic on every run. Give them controlled tools that call approved scripts.

Recommended tools:

```python
class RunScriptTool(BaseTool):
    name = "run_python_script"
    description = "Run an approved project Python script with arguments."

class ReadJsonTool(BaseTool):
    name = "read_json"
    description = "Read a JSON artifact."

class ReadCsvSummaryTool(BaseTool):
    name = "read_csv_summary"
    description = "Summarize rows, columns, missing values, and numeric ranges."

class WriteReportTool(BaseTool):
    name = "write_markdown_report"
    description = "Write a Markdown report from provided structured results."
```

Restrict script execution to approved scripts under:

```text
activity/scripts/
```

This keeps the CrewAI system controlled and reproducible.

## Implementation Milestones

### Milestone 1: Script-First Pipeline

Build scripts that can run without CrewAI:

- Validate inputs.
- Build features.
- Train candidate tabular models.
- Evaluate leaderboard.
- Generate blinded predictions.
- Make submission CSV.
- Write report.

This is the most important milestone. CrewAI should not be added before the script contracts are stable.

### Milestone 2: CrewAI Wrapper

Create agents and tasks that call the stable scripts:

- `agents.py`
- `tasks.py`
- `tools.py`
- `crew.py`
- YAML configs for agents and tasks.

### Milestone 3: Add Model Families

Add or wrap:

- XGBoost.
- Chemprop.
- Chemprop plus LF features.
- Simple ensemble models.

### Milestone 4: Add Review Gates

Block submission generation if:

- Row counts are wrong.
- Predictions are missing.
- Values are not finite.
- Model artifacts are missing.
- Leakage risks are detected.

### Milestone 5: Submission Packaging

Generate a complete final package:

```text
submission.csv
submission_predictions_detailed.csv
run_config.json
model_leaderboard.csv
submission_review.md
method_report.md
```

## Practical Recommendation

Use CrewAI as an orchestration layer, not as the modeling engine.

Durable assets should remain:

- Python scripts.
- CSV feature tables.
- JSON configs.
- Model artifacts.
- Markdown reports.
- Submission CSV files.

CrewAI should coordinate these assets and enforce workflow discipline. This gives automation without losing scientific reproducibility.

For the current project stage, the best next step is to implement the script-first `pEC50` pipeline and only then wrap it with CrewAI agents.
