# Benchmark Comparison: Random Forest vs. Baseline (Logistic Regression)

Both models were evaluated on the same held-out test set. Values below are taken directly from the executed `Week6&7_Models.ipynb` run.

---

## 1. Overall benchmark (macro-averaged)

| Model | Accuracy | Precision (macro) | Recall (macro) | F1 (macro) |
|---|---|---|---|---|
| **Baseline: Logistic Regression** | 0.6799 | 0.57 | 0.48 | 0.5064 |
| **Complex model: Random Forest** | 0.5276 | 0.3710 | 0.4255 | 0.3651 |

**Weighted-average view** (accounts for class support, e.g. the large ESI 3
group), for reference:

| Model | Precision (weighted) | Recall (weighted) | F1 (weighted) |
|---|---|---|---|
| **Baseline: Logistic Regression** | 0.68 | 0.68 | 0.67 |
| **Complex model: Random Forest** | 0.6157 | 0.5276 | 0.5245 |

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

