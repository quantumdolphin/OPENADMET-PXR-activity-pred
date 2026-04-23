## Transductive vs Inductive — in the context of this PXR work

The two terms come from machine-learning theory and are about **which molecules the model will be asked to predict on at test time**. They do not describe the model, they describe the *evaluation setup*.

## 1. The Minimal Definition

- **Transductive setting.** At training time, the model sees (or at least is aware of) the *identities* — and usually the low-fidelity labels — of the exact test molecules. The goal is just to predict their unknown high-fidelity labels.
- **Inductive setting.** At test time, the model is given molecules that were **not part of any training pool**, including the LF screen. Their LF labels don't exist because the molecules were never assayed.

A shorthand:
- Transductive = "predict HF for molecules I've already seen at LF."
- Inductive = "predict HF for molecules that didn't exist during the LF campaign."

## 2. Why the Distinction Only Matters in Multi-Fidelity

In a *single*-fidelity setup, everything is inductive by default: a molecule is in train or in test; you never see the test molecule's label until evaluation. The transductive/inductive split becomes *interesting* only when you have **two sources of information about a molecule** — here, an LF measurement and an HF measurement.

That creates three possible states per molecule:

| State                     | LF available? |         HF available?         |
| ------------------------- | :-----------: | :---------------------------: |
| Transductive training row |       ✓       |               ✓               |
| **Transductive test row** |     **✓**     | **unknown (to be predicted)** |
| **Inductive test row**    |     **✗**     | **unknown (to be predicted)** |
The transductive test row is *easier* because you already have a noisy, cheap observation of the answer (the LF signal) — you just need to refine it to HF. The inductive test row is strictly harder because you only have the SMILES.

## 3. What This Looks Like in Buterez et al. 2024

The paper uses this distinction surgically. Consider their four transfer-learning strategies applied to a test molecule:

| Strategy                            |   Needs an LF measurement at test time?    |              Works inductively?               |
| ----------------------------------- | :----------------------------------------: | :-------------------------------------------: |
| 1. Raw LF label as a feature        |                    Yes                     | **No** — there is no LF label to concatenate. |
| 2. LF **model's predicted** label   | No (the LF GNN can predict it from SMILES) |                      Yes                      |
| 3. LF **model's embedding**         |                     No                     |                      Yes                      |
| 4. Pre-train on LF, fine-tune on HF |                     No                     |                      Yes                      |
Strategy 1 is only a *transductive* trick — at deployment you usually don't have it. Strategies 2–4 work in both regimes because the LF information is *produced by a model from SMILES*, not measured wetly.

The paper also shows that **inductive uplifts are roughly half of transductive uplifts** on most datasets (their Fig. 5B vs Fig. 4). That makes sense: in the transductive case, there is a real experimental LF measurement backing the prediction; in the inductive case, the "LF" is itself a model output and inherits model error.

## 4. How the Two States Appear in the PXR Data

From today's correlation audit:

- HF has 4,140 molecules with `pEC50`.
- LF has 21,014 single-dose rows covering 2,746 unique molecules.
- **2,727 HF molecules have an LF measurement → transductive subset.**
- **1,413 HF molecules have no LF row at all → inductive subset within the training file.**

And critically, the **blinded 513-compound analog test set** is almost entirely inductive: the workflow doc (`OPENADMET_PXR_SCREENING_WORKFLOW_FROM_BLOG.md`) explains those compounds were ordered from Enamine *after* the 63 potent seeds were identified — they were never in the primary screen.

So for the competition:
- You can use the transductive subset to **tune** transfer-learning methods and confirm they work.
- But the leaderboard is scored on an **inductive** test set, so only strategies 2–4 actually earn points.

## 5. Concrete Example

Say your model is asked: *"Predict the pEC50 of molecule X."*

**Transductive path.** X is in the single-concentration CSV. You look up `log2_fc_estimate = 1.8` at 8 µM. Your HF model takes SMILES + `1.8` as features and outputs `pEC50 = 6.3`. This is strategy 1 (or the corresponding augmented variant).

**Inductive path.** X is not in the single-concentration CSV. You only have SMILES. Your LF GNN has been trained to predict `log2_fc_estimate` from SMILES; it outputs `predicted_log2_fc = 1.5`. Your HF model takes SMILES + `1.5` as features and outputs `pEC50 = 6.1`. That's strategy 2.

Or, more robustly: your VGAE was pretrained on LF. At inference you run it on X's graph, get a pooled embedding `z ∈ ℝ^128`, and hand `z` to the HF model (strategy 3) or fine-tune the whole pretrained VGAE on HF (strategy 4).

## 6. Why the Paper's Main Architectural Fix Matters Here

The paper's central empirical finding is that **strategy 3 and strategy 4 fail if the GNN readout is sum/mean/max** — the embeddings or pretrained weights carry LF information that the fixed pooling can't adapt to during HF fine-tuning. Adaptive (Set Transformer) readouts fix this.

For the PXR challenge, that translates to a direct recommendation: since your scored test is inductive, you cannot rely on strategy 1; you need strategies 2/3/4; and those require an adaptive readout to beat the no-augmentation baseline. That's the line of reasoning the synthesis note builds on.

## 7. Common Sources of Confusion

- **Transductive ≠ data leakage.** It does not mean the model saw the *HF* label during training. It means the model saw (or can look up) an *LF* observation of the same molecule. HF labels remain hidden until scoring.
- **A model isn't "transductive" or "inductive."** The *evaluation protocol* is. The same trained VGAE can be evaluated transductively (on molecules that had LF in training) or inductively (on fresh molecules).
- **In graph ML more broadly**, "transductive" sometimes refers to classic node-classification in a single graph (all nodes visible at train time). That's a related but narrower sense — here, every molecule is its own graph, and the relevant question is purely about LF coverage.

## 8. Practical Reporting Rule for pxr_gnn

Whenever a model is scored, split metrics into three buckets and report all three:

1. **Transductive validation/holdout** — HF molecules with a matched LF row. Expect the largest gains from transfer learning.
2. **Inductive validation/holdout** — HF molecules without a matched LF row.
3. **Blinded test** — fully inductive by construction.

If the transductive-to-inductive gap blows up, that's a sign the LF-to-HF pipe is overfitting to transductive convenience and will not generalize to the leaderboard. The paper's Fig. 5B is effectively an instruction to monitor exactly this ratio.
