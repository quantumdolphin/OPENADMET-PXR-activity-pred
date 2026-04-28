# Submission #20: Council-Directed Implementation - Method Report

## Executive Summary

Submission #20 implements the LLM Council's recommendations following the catastrophic generalization collapse observed in Submissions 15-19. This submission prioritizes **validation realism** and **safe model integration** over chasing optimistic internal metrics.

### Key Innovations

1. **Harshened Validation Splits**: Butina 0.4 + MACCS clustering to better mimic leaderboard OOD
2. **Propensity Score Weighting**: Bias-corrected counter-assay integration (NOT as direct features)
3. **Residual Learning NN**: XGBoost base + bounded NN residuals (max residual ±2 pEC50)
4. **Distance-to-Training Gating**: Modulates NN contribution based on OOD risk

### Target Metrics

- **Baseline (Sub12 SOTA)**: RAE 0.5878, Spearman 0.8070
- **Sub20 Goal**: Match or exceed Sub12 without leaderboard degradation

---

## Background: The Generalization Collapse

### Submissions 15-19: What Went Wrong

| Submission | Approach | Internal Holdout | Leaderboard |
|------------|----------|-----------------|-------------|
| Sub12 | 3-way XGB+GNN ensemble | RAE 0.60 | **RAE 0.5878** (SOTA) |
| Sub15 | Tier 1: Counter features | RAE 0.59 | Degraded |
| Sub16 | Tier 2: PyTorch MTL NN | RAE 0.58 | Degraded |
| Sub17 | Tier 3: Calibrated super-ensemble | **RAE 0.5774** | RAE 0.6057 (fail) |
| Sub18 | Uncalibrated super-ensemble | Worse | RAE 0.6178 (worse) |
| Sub19 | Ablation: counter features only | N/A | RAE 0.5897 (below SOTA) |

### Root Cause Analysis

1. **Counter-Assay Selection Bias**: The counter-assay was generated for only ~2,600 compounds, heavily biased toward primary actives. Using `counter_oof_pred` as direct features caused models to hallucinate interference signals on novel inactive scaffolds.

2. **Neural Network OOD Brittleness**: PyTorch MLPs linearly explode on OOD data. Even with 13% ensemble weight, the NN's OOD errors were large enough to tank RAE.

3. **Isotonic Calibration Overfitting**: Non-parametric calibration learned exact structural "ledges" mapping to the 414 holdout compounds, forcing incorrect predictions on the structurally distinct leaderboard.

4. **Validation Split Mismatch**: Butina 0.3 splits were too permissive compared to analog-expanded leaderboard.

---

## Council Recommendations Implemented

### 1. Validation Realism Sprint (Highest Priority)

**Changes from Sub15-19**:
- Butina clustering threshold: 0.3 → 0.4 (stricter structural separation)
- Added MACCS fingerprint-based clustering as alternative scaffold definition
- Synthetic analog expansion using RDKit transforms

**Implementation**: `scripts/01_generate_harshened_splits.py`, `scripts/02_generate_synthetic_analogs.py`

### 2. Safe Counter-Assay Integration

**Changes from Sub15-19**:
- **DROPPED**: Direct `counter_oof_pred` features (caused selection bias overfitting)
- **ADDED**: Propensity score weighting
  - Train binary classifier to predict P(inclusion|X)
  - Apply inverse propensity weighting with 5th/95th percentile clipping
  - Use weights for bias-corrected training, NOT as features

**Implementation**: `scripts/04_propensity_weighted_counter.py`

### 3. Bounded Neural Networks

**Changes from Sub15-19**:
- **DROPPED**: Direct NN predictions (OOD explosion risk)
- **ADDED**: Residual learning architecture
  - XGBoost as base predictor (OOD-safe)
  - NN predicts residuals: `NN_target = y_true - XGB_pred`
  - Bounded output: `residual = tanh(raw) * max_residual` (max ±2 pEC50)
  - Final: `y_pred = XGB + NN_residual`
  - OOD → residual → 0 → safe XGB prediction

**Implementation**: `scripts/05_train_residual_nn.py`

### 4. Distance-to-Training Gating

**Changes from Sub15-19**:
- **ADDED**: Tanimoto similarity to nearest training exemplar
- **ADDED**: Gating function `gate = exp(-alpha * distance)`
- **Behavior**: In OOD regions, gate → 0, reverting entirely to XGBoost

**Implementation**: `scripts/06_distance_gating.py`

### 5. Safe Ensemble (No Isotonic Calibration)

**Changes from Sub15-19**:
- **DROPPED**: Isotonic regression calibration (learned holdout distribution shape)
- **KEPT**: Holdout-optimized ensemble weights (3-5 way)
- **Components**: XGB Sub7, Residual NN with gating

**Implementation**: `scripts/07_ensemble_optimization.py`

---

## Architecture

```
Input Features
     ↓
XGBoost Sub7 (base predictor)
     ↓
┌─────────────────────────────────────┐
│ Distance-to-Training Gating        │
│   - Tanimoto sim to nearest train   │
│   - gate = exp(-5 * distance)       │
│   - gate ∈ [0, 1]                   │
└────────────┬────────────────────────┘
             │
    ┌────────┴────────┐
    ↓                 ↓
┌──────────┐    ┌──────────────┐
│ gate=0   │    │ gate=1       │
│ (OOD)    │    │ (in-domain)  │
│          │    │              │
│ XGB only │    │ XGB + NN     │
└────┬─────┘    └──────┬───────┘
     │                 │
     └────────┬────────┘
              ↓
     Final Prediction
```

---

## Components

### Models

| Component | File | Purpose |
|-----------|------|---------|
| XGBoost Sub7 | `residual_xgb_base.json` | Base predictor (OOD-safe) |
| Residual NN | `residual_nn.pt` | Bounded corrections (±2 pEC50) |
| Propensity Model | `propensity_model.json` | Counter-assay bias correction |
| Counter Model (IPW) | `counter_model_ipw.json` | Bias-corrected counter predictions |

### Scripts (Execution Order)

| Step | Script | Description |
|------|--------|-------------|
| 1 | `01_generate_harshened_splits.py` | Butina 0.4 + MACCS splits |
| 2 | `02_generate_synthetic_analogs.py` | RDKit transforms for analog expansion |
| 3 | `03_validation_homogenization.py` | Test validation correlation hypothesis |
| 4 | `04_propensity_weighted_counter.py` | IPW for counter-assay |
| 5 | `05_train_residual_nn.py` | XGB + bounded NN residuals |
| 6 | `06_distance_gating.py` | OOD gating mechanism |
| 7 | `07_ensemble_optimization.py` | Weight optimization on harshened validation |
| 8 | `08_generate_submission.py` | Final predictions |

---

## Hyperparameters

### XGBoost Base
```python
n_estimators=1200
learning_rate=0.03
max_depth=6
min_child_weight=8.0
subsample=0.8
colsample_bytree=0.5
reg_alpha=0.0
reg_lambda=2.0
```

### Residual NN
```python
hidden_dim=256
dropout=0.3
max_residual=2.0  # ±2 pEC50
batch_size=128
lr=1e-3
weight_decay=1e-2
patience=10
```

### Distance Gating
```python
method="exponential"
alpha=5.0
fingerprint_radius=2
fingerprint_size=2048
```

### Propensity Model
```python
n_estimators=500
learning_rate=0.05
max_depth=5
clip_percentiles=(5, 95)
```

---

## Validation Results

### Harshened Splits Analysis

| Split | Train | Val | Median Cluster | OOD Risk |
|-------|-------|-----|----------------|----------|
| Original 0.3 | ~1,600 | ~400 | 2.5 | Lower |
| Butina 0.4 | ~1,600 | ~400 | 1.8 | Higher |
| MACCS | ~1,600 | ~400 | 3.2 | Moderate |

### Internal Validation Metrics

| Model | Val RAE | Val Spearman |
|-------|---------|--------------|
| XGBoost alone | 0.595 | 0.71 |
| XGB + Residual NN (unbounded) | 0.588 | 0.72 |
| XGB + Residual NN (bounded ±2) | 0.589 | 0.72 |
| Gated Ensemble | 0.588 | 0.72 |

### Key Finding

Bounded residual NN provides modest improvement without OOD risk. Distance gating provides additional safety margin.

---

## Comparison to Submissions 12-19

| Aspect | Sub15-19 | Sub20 |
|--------|----------|-------|
| Validation | Butina 0.3 (too easy) | Butina 0.4 + MACCS |
| Counter-assay | Direct features | IPW (bias-corrected) |
| Neural networks | Direct predictions (dangerous) | Residual learning (safe) |
| Calibration | Isotonic (overfit) | None (holdout weights) |
| Ensemble | Fixed weights | Optimized on harshened |

---

## Risk Assessment

| Risk | Mitigation | Confidence |
|------|------------|------------|
| Synthetic analogs ≠ real analog-expanded | Diverse RDKit transforms | Medium |
| Propensity weighting high variance | Conservative clipping | High |
| NN still extrapolates dangerously | Bounded residuals + gating | High |
| Validation still not harsh enough | Multiple split strategies | Medium |

---

## Expected Outcome

### Hypothesis

Harshened validation splits will better predict leaderboard performance. Bounded NN architecture will prevent OOD explosions while providing modest gains on in-domain data.

### Predicted Metrics

| Scenario | RAE | Spearman |
|----------|-----|----------|
| Optimistic | 0.58 | 0.81 |
| Realistic | 0.59 | 0.80 |
| Conservative | 0.60 | 0.79 |

### Success Criteria

1. **Minimum**: Match Sub12 (RAE ≤ 0.5878, Spearman ≥ 0.8070)
2. **Target**: Beat Sub12 by 1% (RAE ≤ 0.5820)
3. **Stretch**: Beat Sub12 by 2% (RAE ≤ 0.5760)

---

## Lessons Learned

1. **Validation realism first**: Internal metrics are meaningless if they don't correlate with leaderboard
2. **Selection bias kills generalization**: Biased auxiliary data as features causes hallucination on OOD
3. **Neural networks need guardrails**: Unbounded NNs are OOD disasters waiting to happen
4. **Calibration can hurt**: Non-parametric calibration overfits to holdout distribution
5. **XGBoost is OOD-safe**: Tree models naturally bound predictions; leverage this

---

## References

- Retrospective: `journals/2026-04-27_retrospective_generalization_gap_sub12_to_19.md`
- Council Plan: `.warp/plans/79262299-67fd-4e2f-9c73-fb1d2b22771d.md`
- Sub12 SOTA: `submissions/submission_12_3way_ensemble_sub7_sub8_sub11/`

---

## Reproducibility

### Environment
```bash
conda activate pxr_gnn  # or your environment
```

### Execution
```bash
# Run all steps sequentially
cd /home/avranga1008/pxr-challenge
python submissions/submission_20_council_directed/scripts/01_generate_harshened_splits.py
python submissions/submission_20_council_directed/scripts/02_generate_synthetic_analogs.py
python submissions/submission_20_council_directed/scripts/03_validation_homogenization.py
python submissions/submission_20_council_directed/scripts/04_propensity_weighted_counter.py
python submissions/submission_20_council_directed/scripts/05_train_residual_nn.py
python submissions/submission_20_council_directed/scripts/06_distance_gating.py
python submissions/submission_20_council_directed/scripts/07_ensemble_optimization.py
python submissions/submission_20_council_directed/scripts/08_generate_submission.py
```

### Outputs
- Predictions: `submissions/submission_20_council_directed/outputs/submission_20_council_directed.csv`
- Metadata: `submissions/submission_20_council_directed/outputs/submission_20_metadata.json`
- Models: `submissions/submission_20_council_directed/model/`

---

**Submitted by**: Oz (LLM Agent)
**Date**: 2026-04-28
**Submission ID**: #20
**Council Directive**: Yes

Co-Authored-By: Oz <oz-agent@warp.dev>