# Benchmark Comparison: Random Forest vs. Baseline (Logistic Regression)

Both models were evaluated on the same held-out test set. Values below are taken directly from the executed `Week6&7_Models.ipynb` run.

---

## 1. Six-axis benchmark (macro-averaged, plus cost and interpretability)

| Model | Accuracy | Precision (macro) | Recall (macro) | F1 (macro) | Training time (s) | Inference time (ms/pred) | Interpretability |
|---|---|---|---|---|---|---|---|
| **Baseline: Logistic Regression** | 0.6799 | 0.572 | 0.4792 | 0.5064 | 35.5007 | 0.0016 | **High** — coefficients can be read directly; a single prediction can be explained to a clinician in under a minute |
| **Complex model: Random Forest** | 0.5276 | 0.371 | 0.4255 | 0.3651 | 20.5167 | 0.0386 | **Medium** — `feature_importances_` gives an aggregate ranking only; explaining one specific patient's prediction in under a minute is not possible without an additional method such as SHAP |

**Weighted-average view** (accounts for class support, e.g. the large ESI 3
group), for reference:

| Model | Precision (weighted) | Recall (weighted) | F1 (weighted) |
|---|---|---|---|
| **Baseline: Logistic Regression** | 0.68 | 0.68 | 0.67 |
| **Complex model: Random Forest** | 0.6157 | 0.5276 | 0.5245 |

**Reading the cost columns:** Logistic Regression takes longer to train here (35.5 s vs. 20.5 s) because it iterates over all 262 model inputs to convergence (`max_iter=1000`), whereas Random Forest's 300 trees each train quickly on a bootstrap sample. The relationship flips at inference: Logistic Regression scores a single patient in about 0.0016 ms, roughly 24 times faster than Random Forest's 0.0386 ms, because Random Forest must query and combine all 300 trees for every prediction. In a live ED, inference time is paid once per patient, forever, so it is the more operationally relevant of the two costs — and Logistic Regression wins on both.

**Reading the interpretability column:** Logistic Regression's coefficients give a direct, per-patient explanation — the sign and size of each coefficient shows exactly how that input pushed the prediction, which passes the "explain it to a clinician in under a minute" test. Random Forest's `feature_importances_` only ranks which inputs matter *on average* across all 300 trees; it cannot say why any single patient was flagged, so it does not pass that same test without adding SHAP.

---

## 2. Per-class breakdown (ESI 1–5)

| ESI level | Metric | Baseline:  Logistic Regression | Complex model: Random Forest | Support (test set) |
|---|---|---|---|---|
| **ESI 1** | Precision | 0.40 | 0.0000 | 16 |
| **ESI 1** | Recall | 0.25 | 0.0000 | 16 |
| **ESI 1** | F1 | 0.31 | 0.0000 | 16 |
| **ESI 2** | Precision | 0.73 | 0.5710 | 3,585 |
| **ESI 2** | Recall | 0.62 | 0.7291 | 3,585 |
| **ESI 2** | F1 | 0.67 | 0.6405 | 3,585 |
| **ESI 3** | Precision | 0.68 | 0.7536 | 5,402 |
| **ESI 3** | Recall | 0.76 | 0.3545 | 5,402 |
| **ESI 3** | F1 | 0.72 | 0.4822 | 5,402 |
| **ESI 4** | Precision | 0.63 | 0.3524 | 1,779 |
| **ESI 4** | Recall | 0.61 | 0.6734 | 1,779 |
| **ESI 4** | F1 | 0.62 | 0.4626 | 1,779 |
| **ESI 5** | Precision | 0.43 | 0.1779 | 243 |
| **ESI 5** | Recall | 0.14 | 0.3704 | 243 |
| **ESI 5** | F1 | 0.22 | 0.2403 | 243 |
