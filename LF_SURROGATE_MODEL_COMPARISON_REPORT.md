# Low-Fidelity Surrogate Model Comparison

This report compares three model families trained to predict the low-fidelity PXR single-concentration screening signals:

- `lf_signal_8uM`
- `lf_signal_33uM`

The goal of these models is not to directly predict final PXR `pEC50`. The goal is to learn a molecular surrogate for the low-fidelity screening assay so that predicted low-fidelity features can later be added to the downstream dose-response model.

## Executive Summary

The strongest practical default remains the original Chemprop v2 multitask regression model because it gives the best `8uM` performance and strong `33uM` performance with the simplest deployment path. CheMeleon foundation initialization modestly improved `33uM` metrics but weakened `8uM`, so it did not improve the overall two-target surrogate. XGBoost with RDKit descriptors plus Morgan fingerprints was useful as a fast, interpretable baseline, but tuning did not materially improve held-out performance.

The most important result is that hyperparameter tuning must be judged by held-out test behavior, not by validation RMSE alone. The XGBoost Optuna search found some lower-validation-RMSE models, but those gains were very small and did not clearly transfer to the test set.

## Test Metrics

| target | model | selection | n | rmse | mae | r2 | pearson | spearman |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| lf_signal_33uM | XGBoost baseline | fixed parameters | 970 | 0.4237 | 0.3211 | 0.4311 | 0.6577 | 0.6802 |
| lf_signal_33uM | XGBoost Optuna val-best | lowest validation RMSE | 970 | 0.4255 | 0.3232 | 0.4263 | 0.6544 | 0.6771 |
| lf_signal_33uM | Chemprop + CheMeleon | test checkpoint | 953 | 0.4385 | 0.3287 | 0.4537 | 0.6758 | 0.6894 |
| lf_signal_33uM | XGBoost Optuna penalized | lowest penalized objective | 970 | 0.4389 | 0.3363 | 0.3895 | 0.6292 | 0.6496 |
| lf_signal_33uM | Chemprop v2 | test checkpoint | 953 | 0.4414 | 0.3328 | 0.4465 | 0.6695 | 0.6865 |
| lf_signal_8uM | Chemprop v2 | test checkpoint | 1073 | 0.3002 | 0.2111 | 0.4811 | 0.6939 | 0.6623 |
| lf_signal_8uM | Chemprop + CheMeleon | test checkpoint | 1073 | 0.3040 | 0.2098 | 0.4679 | 0.6894 | 0.6586 |
| lf_signal_8uM | XGBoost Optuna val-best | lowest validation RMSE | 1070 | 0.3207 | 0.2222 | 0.4176 | 0.6506 | 0.6306 |
| lf_signal_8uM | XGBoost baseline | fixed parameters | 1070 | 0.3210 | 0.2210 | 0.4165 | 0.6508 | 0.6336 |
| lf_signal_8uM | XGBoost Optuna penalized | lowest penalized objective | 1070 | 0.3263 | 0.2256 | 0.3970 | 0.6369 | 0.6225 |

## Visual Comparison

![Test RMSE comparison](test_rmse_comparison.png)

![Test R2 comparison](test_r2_comparison.png)

![Test Spearman comparison](test_spearman_comparison.png)

![Observed vs predicted grid](observed_vs_predicted_grid.png)

The bar plots show aggregate performance by target. The observed-vs-predicted plots show whether predictions track the continuous assay signal across the range, not just whether the average error is small.

Important comparison caveat: Chemprop and XGBoost were both evaluated on held-out test splits, but the row counts differ slightly because Chemprop generated its saved split internally while XGBoost used the later saved scikit-learn split. Therefore, the metrics are good model-level estimates, but they are not a perfectly paired per-compound comparison. This matters most for `lf_signal_33uM`, where XGBoost has a lower RMSE on its split while Chemprop/CheMeleon have stronger R2/correlation behavior on their split.

## Data Used

The common starting table was:

```text
activity/data/features/single_dose_chemprop/chemprop_lf_regression_8_33.csv
```

Columns:

```text
SMILES
lf_signal_8uM
lf_signal_33uM
```

This table has 10,835 compounds. The target columns are not complete:

```text
lf_signal_8uM missing values: 83
lf_signal_33uM missing values: 1308
```

Missing values were not treated as test labels. They are unlabeled target entries. For multitask Chemprop, missing target values can be masked so that a row can still contribute to another target if that target is present. For the two independent XGBoost models, rows with a missing value for the target being trained were dropped for that target only.

## Train, Validation, And Test Splits

The modeling used an 80/10/10 split:

```text
train: 80%
validation: 10%
test: 10%
random seed: 42
```

The split has different roles:

- Training set: used to fit model parameters.
- Validation set: used to monitor model selection, early stopping, or hyperparameter choice.
- Test set: held out until evaluation, used as the final estimate of generalization.

For XGBoost, the saved global split counts were:

```text
train: 8667
validation: 1084
test: 1084
```

After dropping missing target values independently for each target, XGBoost used:

```text
lf_signal_8uM: train 8601, validation 1081, test 1070
lf_signal_33uM: train 7590, validation 967, test 970
```

Chemprop saved its split files here:

```text
activity/outputs/chemprop_lf_regression_8_33/train.csv
activity/outputs/chemprop_lf_regression_8_33/val.csv
activity/outputs/chemprop_lf_regression_8_33/test.csv
activity/outputs/chemprop_lf_regression_8_33/splits.json
```

XGBoost saved its feature matrix and split files here:

```text
activity/outputs/xgboost_lf_8_33/lf_xgboost_features_8_33.csv
activity/outputs/xgboost_lf_8_33/splits.csv
activity/outputs/xgboost_lf_8_33/splits.json
```

## Method 1: Chemprop V2 Multitask Regression

Chemprop was trained as a multitask graph neural network:

```text
SMILES -> lf_signal_8uM, lf_signal_33uM
```

This model learns directly from molecular graphs. It does not require manually generated fingerprints or descriptors. The training used the two target columns and excluded `lf_signal_99uM`.

Important artifacts:

```text
activity/outputs/chemprop_lf_regression_8_33/model_0/best.pt
activity/outputs/chemprop_lf_regression_8_33/model_0/test_predictions.csv
activity/outputs/chemprop_lf_regression_8_33/model_0/trainer_logs/version_0/metrics.csv
activity/outputs/chemprop_lf_regression_8_33/config.toml
activity/outputs/chemprop_lf_regression_8_33/CHEMPROP_LF_REGRESSION_RUN_REPORT.md
```

The best validation checkpoint was selected from the training run. The final metrics in this report were recomputed from:

```text
observed values: activity/outputs/chemprop_lf_regression_8_33/test.csv
predictions: activity/outputs/chemprop_lf_regression_8_33/model_0/test_predictions.csv
```

## Method 2: Chemprop With CheMeleon Foundation Initialization

The CheMeleon experiment used Chemprop's foundation-model initialization:

```text
--from-foundation chemeleon
```

The CheMeleon weights were downloaded by Chemprop and used to initialize the model before fine-tuning on the same LF targets. This increased model size and runtime, but did not improve held-out performance here.

Important artifacts:

```text
activity/outputs/chemprop_lf_regression_8_33_chemeleon/model_0/best.pt
activity/outputs/chemprop_lf_regression_8_33_chemeleon/model_0/test_predictions.csv
activity/outputs/chemprop_lf_regression_8_33_chemeleon/model_0/trainer_logs/version_0/metrics.csv
activity/outputs/chemprop_lf_regression_8_33_chemeleon/config.toml
```

Interpretation: CheMeleon may still be useful in other settings, and here it did slightly improve `lf_signal_33uM`. It did not improve the two-target surrogate overall because `lf_signal_8uM` became worse. The likely reason is that the LF signals are assay-specific, noisy, and tied to the OpenADMET screen; foundation initialization is not guaranteed to help unless the inductive bias matches the downstream task.

## Method 3: XGBoost Baseline

XGBoost was trained as two separate single-target regressors:

```text
SMILES -> lf_signal_8uM
SMILES -> lf_signal_33uM
```

The `drug-ml` conda environment was used:

```text
C:\anaconda3\envs\drug-ml\python.exe
xgboost 3.1.2
rdkit 2025.09.2
sklearn 1.7.2
```

Because `scikit-mol` was not installed in `drug-ml`, the XGBoost pipeline used native RDKit features:

```text
RDKit 2D descriptors: 217
Morgan fingerprint bits: 2048
Total feature columns: 2265
```

The fixed baseline parameters were:

```json
{
  "n_estimators": 800,
  "learning_rate": 0.03,
  "max_depth": 4,
  "min_child_weight": 5.0,
  "subsample": 0.8,
  "colsample_bytree": 0.5,
  "reg_alpha": 0.0,
  "reg_lambda": 2.0,
  "gamma": 0.0,
  "tree_method": "hist"
}
```

Important script and artifacts:

```text
activity/scripts/train_lf_xgboost_8_33.py
activity/outputs/xgboost_lf_8_33/run_config.json
activity/outputs/xgboost_lf_8_33/lf_signal_8uM/model.json
activity/outputs/xgboost_lf_8_33/lf_signal_8uM/pipeline.joblib
activity/outputs/xgboost_lf_8_33/lf_signal_33uM/model.json
activity/outputs/xgboost_lf_8_33/lf_signal_33uM/pipeline.joblib
```

## Method 4: XGBoost Hyperparameter Search

Optuna was used to tune XGBoost separately for each target. The search varied:

```text
n_estimators
learning_rate
max_depth
min_child_weight
subsample
colsample_bytree
reg_alpha
reg_lambda
gamma
```

The tuning script was:

```text
activity/scripts/tune_lf_xgboost_8_33_optuna.py
```

The first batch used:

```text
20 trials per target
TPESampler
random seed 42
same saved train/validation/test split as the XGBoost baseline
```

### Penalized Objective

The initial Optuna objective was not pure validation RMSE. It used:

```text
objective = validation_RMSE + 0.25 * max(0, validation_RMSE - training_RMSE)
```

This penalizes models that look much better on the training set than on validation. The intention was to reduce overfitting and prefer models that generalize more smoothly.

That objective selected:

```text
lf_signal_8uM best penalized trial: 2
lf_signal_33uM best penalized trial: 5
```

However, the selected penalized models were worse on the held-out test set than the fixed XGBoost baseline. This is useful evidence: the penalty was too conservative for this problem, or the search space/trial count was too small to find a better model under that objective.

Important artifacts:

```text
activity/outputs/xgboost_lf_8_33_optuna/run_config.json
activity/outputs/xgboost_lf_8_33_optuna/lf_signal_8uM/trials.csv
activity/outputs/xgboost_lf_8_33_optuna/lf_signal_33uM/trials.csv
```

### Best Validation-RMSE Candidates

Because the penalized objective was conservative, we also refit the lowest-validation-RMSE candidate from the same Optuna trials:

```text
activity/scripts/evaluate_lf_xgboost_optuna_val_best.py
```

Selected validation-best trials:

```text
lf_signal_8uM trial: 10
lf_signal_33uM trial: 19
```

These models slightly improved validation RMSE but still did not materially beat the original XGBoost baseline on the test set.

Important artifacts:

```text
activity/outputs/xgboost_lf_8_33_optuna_val_best/run_config.json
activity/outputs/xgboost_lf_8_33_optuna_val_best/candidate_comparison.csv
activity/outputs/xgboost_lf_8_33_optuna_val_best/lf_signal_8uM/model.json
activity/outputs/xgboost_lf_8_33_optuna_val_best/lf_signal_33uM/model.json
```

## Interpretation

Chemprop remains the safest current LF surrogate because it has the strongest `8uM` performance and a simple multitask graph-based deployment path. CheMeleon is competitive and slightly stronger on `33uM`, but not across both targets. XGBoost remains useful because it gives fast feature importances, simpler debugging, and an independent check on whether molecular fingerprints and descriptors contain enough signal. In this experiment, they do contain signal, but tuning did not make XGBoost a clearly better surrogate.

CheMeleon did not improve performance in this run. That does not prove foundation initialization is bad. It only shows that this specific LF screening surrogate, with these settings and CPU-friendly training constraints, did not benefit from it.

The most important modeling lesson is that validation optimization alone is not enough. The XGBoost validation-best candidates looked slightly better on validation, but the held-out test set showed only marginal or negative improvement. For downstream use, use the model that best balances test performance, stability, and ease of deployment.

## Practical Recommendation

Use Chemprop v2 as the primary LF surrogate for generating downstream features:

```text
predicted_screen_signal_8uM
predicted_screen_signal_33uM
```

Keep XGBoost baseline predictions as optional ensemble/control features:

```text
xgb_predicted_screen_signal_8uM
xgb_predicted_screen_signal_33uM
```

Do not replace Chemprop with the tuned XGBoost models based on the current evidence.

## Reproducibility Map

Input table:

```text
activity/data/features/single_dose_chemprop/chemprop_lf_regression_8_33.csv
```

Chemprop run:

```text
activity/outputs/chemprop_lf_regression_8_33
```

CheMeleon run:

```text
activity/outputs/chemprop_lf_regression_8_33_chemeleon
```

XGBoost baseline:

```text
activity/scripts/train_lf_xgboost_8_33.py
activity/outputs/xgboost_lf_8_33
```

XGBoost Optuna:

```text
activity/scripts/tune_lf_xgboost_8_33_optuna.py
activity/outputs/xgboost_lf_8_33_optuna
```

XGBoost validation-best refit:

```text
activity/scripts/evaluate_lf_xgboost_optuna_val_best.py
activity/outputs/xgboost_lf_8_33_optuna_val_best
```

This report and its generated summary files:

```text
activity/outputs/lf_surrogate_model_comparison/LF_SURROGATE_MODEL_COMPARISON_REPORT.md
activity/outputs/lf_surrogate_model_comparison/model_comparison_metrics.csv
activity/outputs/lf_surrogate_model_comparison/model_comparison_predictions.csv
```
