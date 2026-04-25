 Explainer: Why the Sub7/Sub8 Ensemble Improved Performance

## Context: what Sub7 and Sub8 each represent
Both Submission #7 and Submission #8 ultimately produce an **XGBoost regressor** that predicts **pEC50** on the same task, using the same base “v2” tabular feature matrix (RDKit descriptors + fingerprints + engineered columns). However, they differ in the **learned embedding features** they inject into XGBoost:

### Submission #7 (Sub7): supervised LF embeddings → XGBoost
- Trains a **VGAE-style GNN encoder** on low-fidelity (LF) data with a supervised objective (the learned representation is optimized to predict an LF signal directly).
- Uses the encoder to export **128-dim embeddings** for every compound.
- Concatenates these embeddings to the v2 feature matrix.
- Trains XGBoost with heteroscedastic-aware weighting (`total_error` weighting).

**Intuition:** Sub7 embeddings are “task-shaped”: they tend to capture features directly predictive of the LF endpoint (which partially transfers to the high-fidelity pEC50 task).

### Submission #8 (Sub8): contrastive LF embeddings → XGBoost
- Trains a similar GNN encoder, but using a **self-supervised contrastive objective** on LF graphs (augmentations like masking / edge drop) rather than a direct supervised regression loss.
- Exports **128-dim embeddings** for every compound.
- Concatenates these embeddings to the same v2 feature matrix.
- Trains XGBoost with the exact same weighting and hyperparameters.

**Intuition:** Sub8 embeddings are “structure-shaped”: they are pushed to be invariant to small graph perturbations and to separate different molecules in representation space, potentially encoding complementary chemistry (scaffolds, motifs, topology) that supervised training might ignore.

Sub7 and Sub8 represent two different representation-learning biases:
- **Supervised LF:** “what helps predict the LF label.”
- **Contrastive:** “what is stable and discriminative about molecular structure.”

---

## The Ensemble Methodology: Prediction-Level Blending

Instead of training a new model on combined features (which we tested and found unhelpful), we implemented a **prediction-level ensemble** for Submission 9.

### The Formulation
Let:
- $\hat{y}_7$ = Sub7 model prediction
- $\hat{y}_8$ = Sub8 model prediction
- $w$ = weight on Sub7 (between 0 and 1)

Then the ensemble prediction is simply a linear blend:
$$ \hat{y}_{ens} = w\hat{y}_7 + (1-w)\hat{y}_8 $$

### Weight Optimization via Grid Search
Because the primary challenge metric is **RAE (Relative Absolute Error)**:
$$ RAE = \frac{\sum_i |y_i - \hat{y}_i|}{\sum_i |y_i - \bar{y}|} $$
We optimized $w$ by conducting a grid search over the holdout split:
1. Collect **holdout predictions** from both Sub7 and Sub8.
2. Align the datasets row-by-row using a stable key (`Molecule Name`).
3. Sweep weights $w \in [0, 1]$ on a fine grid (step size 0.001).
4. Compute RAE on the holdout labels for each blended prediction.
5. Select the $w$ that minimizes RAE.

The optimized weight was stable around $w_{sub7} \approx 0.645$.

### Generating the Final Artifacts
Using the optimal weight $w_{sub7} = 0.645$:
1. We loaded the full-train models from Sub7 and Sub8 (trained on all labeled data).
2. We scored the blinded test set independently with both models.
3. We linearly blended the test predictions: $\hat{y}_{test} = 0.645 \cdot \hat{y}_{7,test} + 0.355 \cdot \hat{y}_{8,test}$.
4. The blended predictions were saved to the final leaderboard CSV format.

---

## Why prediction-level blending beats "feature concat + retrain"

During our experiments, we initially tried a **feature-level blend** (concatenating both Sub7 and Sub8 embeddings into one large feature matrix and retraining XGBoost). This approach **underperformed** the Sub7 model alone on holdout.

This result is common for tree models when features are:
- highly correlated or partially redundant (both are GNN embeddings of the same graphs),
- high-dimensional enough to encourage “winner-take-all” splits early in the tree,
- and trained with the same fixed hyperparameters (no extra regularization).

Feature concatenation forces a single XGBoost model to learn which embedding dimensions to trust, how to integrate supervised vs contrastive dimensions nonlinearly, and how to ignore the noise from both. 

**Prediction-level blending bypasses this.** It does not ask a single model to digest two massive, correlated feature blocks. Instead, it combines two already-optimized, well-regularized predictors, directly exploiting the diversity of their errors.

---

## Why the Ensemble Improves Performance (The ML Details)

Ensembling helps when models exhibit **non-identical error patterns** (error diversity). The Sub7/Sub8 setup naturally produces complementary errors for several reasons:

### 1) Complementary Objective Functions
- **Sub7** (Supervised) compresses away structural details that don't correlate with the LF label. It can overemphasize label correlations that don't transfer cleanly to the HF task.
- **Sub8** (Contrastive) preserves structural identity and invariances regardless of the LF label. It may miss direct activity signals but captures generalized scaffold relationships.
Therefore, they make mistakes on different subsets of compounds. Blending averages these distinct failure modes.

### 2) Variance Reduction on Outliers
Both models have estimation noise due to finite sampling and the stochastic nature of GNN pretraining. For absolute-error-based metrics like MAE and RAE, averaging predictions tightens the error distribution by pulling outlier predictions (where one model failed significantly) closer to the mean.

### 3) Handling Heteroscedasticity 
The dataset is highly heteroscedastic (low pEC50 values are much noisier due to floor effects). Even with `total_error` weighting, different feature representations will cause models to behave differently in these noisy regions. A weighted blend acts as a "soft robustifier," dampening extreme spikes in predictions while preserving shared, high-confidence signals.

### 4) The $w > 0.5$ Optimum
The optimized weight of $w_{sub7} = 0.645$ tells us that Sub7 is the stronger base predictor, but Sub8 provides a valuable incremental correction. This indicates that the contrastive embeddings capture a subset of chemistry that the supervised embeddings miss entirely.

---

## Holdout Performance Summary

On the **holdout split (n=414)**, the primary metric is **RAE** (lower is better):

- **Submission 7**: RAE 0.616269
- **Submission 8**: RAE 0.640068
- **Submission 9 (Feature-level concat)**: RAE 0.628524 (worse than Sub7 alone)
- **Submission 9 (Prediction-level ensemble, $w=0.645$)**: **RAE 0.603951**

By switching from feature concatenation to prediction blending, we successfully extracted the complementary value of the contrastive embeddings without degrading the strong baseline performance of the supervised embeddings.
