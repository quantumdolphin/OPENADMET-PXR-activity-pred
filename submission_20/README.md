# Submission #20: Council-Directed Implementation

## Overview

Submission #20 implements the LLM Council's recommendations following the generalization collapse observed in Submissions 15-19. This submission focuses on **validation realism** and **safe model integration** rather than chasing optimistic internal metrics.

## Goals

1. **Fix Validation Mismatch**: Replace permissive Butina 0.3 splits with harshened validation that correlates with leaderboard performance
2. **Safe Counter-Assay Integration**: Use propensity score weighting instead of direct biased features
3. **Bounded Neural Networks**: Implement residual learning and distance gating to prevent OOD extrapolation disasters

## Submission Structure

```
submission_20_council_directed/
├── scripts/
│   ├── 01_generate_harshened_splits.py      # Butina 0.4 + MACCS clustering
│   ├── 02_generate_synthetic_analogs.py      # RDKit transforms for analog expansion
│   ├── 03_validation_homogenization.py     # Test validation correlation hypothesis
│   ├── 04_propensity_weighted_counter.py   # IPW for counter-assay bias correction
│   ├── 05_train_residual_nn.py             # XGBoost base + NN residuals
│   ├── 06_distance_gating.py               # OOD gating mechanism
│   ├── 07_ensemble_optimization.py         # Final ensemble weight optimization
│   └── 08_generate_submission.py           # Generate final predictions
├── features/                                 # Generated feature matrices
├── model/                                    # Trained model artifacts
├── outputs/                                  # Predictions and evaluation results
└── docs/
    ├── council_recommendations.md            # Original council guidance
    ├── validation_analysis.md                # Validation homogenization results
    ├── ablation_studies.md                   # Component ablation results
    └── METHOD_REPORT.md                      # Final submission report
```

## Execution Plan

### Phase 1: Validation Realism (Week 1)
- [ ] Generate harshened splits (Butina 0.4, MACCS clustering)
- [ ] Create synthetic analog-expanded datasets
- [ ] Run validation homogenization experiment
- [ ] Establish correlation between internal and leaderboard metrics

### Phase 2: Safe Counter-Assay Integration (Week 2)
- [ ] Implement propensity score weighting
- [ ] Train propensity model to predict P(inclusion|X)
- [ ] Apply inverse weighting with percentile clipping
- [ ] Evaluate on harshened validation splits

### Phase 3: Bounded Neural Networks (Week 3)
- [ ] Train XGBoost base model
- [ ] Implement residual learning NN
- [ ] Add distance-to-training gating
- [ ] Integrate OOD-aware early stopping

### Phase 4: Final Ensemble (Week 4)
- [ ] Combine safe components
- [ ] Optimize ensemble weights
- [ ] Generate submission predictions
- [ ] Document methodology

## Key Differences from Sub15-19

| Aspect | Sub15-19 | Sub20 (Council-Directed) |
|--------|----------|---------------------------|
| Validation | Butina 0.3 (too permissive) | Butina 0.4 + MACCS + analog expansion |
| Counter-assay | Direct features (`counter_oof_pred`) | Propensity score weighting |
| Neural networks | Direct predictions (OOD explosion) | Residual learning + gating |
| Ensemble | Fixed weights + isotonic calibration | Holdout-optimized, no calibration |
| Success metric | Internal RAE | Leaderboard correlation |

## References

- Retrospective: `journals/2026-04-27_retrospective_generalization_gap_sub12_to_19.md`
- Council Plan: `.warp/plans/79262299-67fd-4e2f-9c73-fb1d2b22771d.md`
- Sub12 SOTA: `submissions/submission_12_3way_ensemble_sub7_sub8_sub11/`
