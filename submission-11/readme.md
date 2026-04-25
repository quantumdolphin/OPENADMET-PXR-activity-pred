# Method Report: 3-Way Ensemble (Sub7 + Sub8 + Sub11)

## Summary
This submission combines three previously trained models at the **prediction level**:

1. **Submission 7** — XGBoost trained on classical cheminformatics features plus **supervised low-fidelity (LF) graph embeddings**.
2. **Submission 8** — XGBoost trained on the same classical features plus **contrastive self-supervised LF graph embeddings**.
3. **Submission 11** — end-to-end **RWPE-enhanced VGAE/Set Transformer GNN**, where node features were augmented with **Random Walk Positional Encodings (RWPE)** before low-fidelity pretraining and high-fidelity fine-tuning.

The ensemble does **not** retrain any base model. Instead, it averages their blinded-test predictions using optimized scalar weights.

## Ensemble Architecture
Let:
- $\hat{y}_7$ = prediction from Submission 7
- $\hat{y}_8$ = prediction from Submission 8
- $\hat{y}_{11}$ = prediction from Submission 11

The final ensemble prediction is:

$$
\hat{y}_{ens} = 0.55\hat{y}_7 + 0.28\hat{y}_8 + 0.17\hat{y}_{11}
$$

### Why these three models?
- **Sub7** is the strongest single hybrid transfer model on internal holdout.
- **Sub8** provides a second learned embedding space with a different self-supervised bias.
- **Sub11** is individually weaker, but introduces a distinct **global structural prior** through RWPE, lowering residual correlation enough to improve the ensemble.

## Weight Selection
The weights were tuned on the internal **Butina holdout split (n=414)** to minimize the **primary challenge metric, RAE**:

$$
RAE = \frac{\sum_i |y_i - \hat{y}_i|}{\sum_i |y_i - \bar{y}|}
$$

A coarse 3-way simplex search found the best holdout blend at approximately:
- **w7 = 0.55**
- **w8 = 0.28**
- **w11 = 0.17**

These weights outperformed:
- each single model,
- the equal-weight 3-way average,
- and the earlier best 2-way Sub7/Sub8 blend.

## Holdout Performance
### Base models
- **Submission 7:** RAE **0.616269**, MAE 0.519814, RMSE 0.748785
- **Submission 8:** RAE **0.640068**, MAE 0.539888, RMSE 0.768362
- **Submission 11:** RAE **0.678305**, MAE 0.572141, RMSE 0.798801

### Ensemble comparisons
- **Best previous 2-way blend (Sub7/Sub8, 0.645 / 0.355):**  
  RAE **0.603951**, MAE 0.509424, RMSE 0.737068, R² 0.537443, Spearman 0.695531

- **Equal-weight 3-way average:**  
  RAE **0.602028**, MAE 0.507802, RMSE 0.730948, R² 0.545092, Spearman 0.698919

- **Optimized 3-way ensemble (0.55 / 0.28 / 0.17):**  
  **RAE 0.599593**, **MAE 0.505748**, **RMSE 0.729861**, **R² 0.546444**, **Spearman 0.703446**

## Interpretation
The main gain comes from combining:
- the strong supervised-transfer model (Sub7),
- the complementary contrastive-transfer model (Sub8),
- and a smaller contribution from the RWPE-based GNN (Sub11), which adds structurally different errors despite weaker standalone performance.

This is a **prediction-level ensemble**, so it is low-risk operationally:
- no additional training,
- simple reproducibility,
- and straightforward leaderboard deployment.

## Submission Artifact
The upload-ready CSV for this ensemble was generated at:

`activity/outputs/submission_11_rwpe/ensemble_3way/leaderboard_upload.csv`

with checksum:

`activity/outputs/submission_11_rwpe/ensemble_3way/leaderboard_upload.csv.sha256`
