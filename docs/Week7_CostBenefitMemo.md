# Cost-Benefit Memo: Choice of Model for ED Triage (ESI) Prediction

---

**Recommendation, in one sentence:** we recommend continuing to develop Logistic Regression as the working model rather than replacing it with Random Forest, because Random Forest performed worse on every measure we tested this week, including recall for the most urgent patients — but neither model is yet safe to use as a stand-alone triage decision-maker, and this choice does not solve that underlying problem.

---

## 1. Recap: the dataset and the method

We are trying to predict a patient's **ESI** (Emergency Severity Index) level from information available at the front door of the emergency department. ESI runs from **1** (needs immediate, life-saving intervention) to **5** (routine, can safely wait).

- **Data**: 55,121 patients and 225 columns, drawn from `triage_cleaned_v1.csv`.
- **Inputs**: vital signs (heart rate, blood pressure, respiratory rate, oxygen saturation, temperature, glucose), demographics (age, gender, ethnicity, language, marital and employment status, insurance), and flags for the patient's presenting complaint.
- **Deliberately excluded**: administrative fields (which department, how the patient arrived, time of arrival) and outcome fields (what happened after triage). The outcome fields are only known *after* a clinician has already triaged the patient, so including them would let the model "cheat" by seeing the answer in advance.
- **Split**: 80% training (44,096 patients) and 20% test (11,025 patients), split so that the proportion of each ESI level is the same in both groups. This split, and the random seed used to create it (42), is fixed and reused for every model below, so the comparison is like-for-like.
- **The one fact that matters most**: the test set contains only **16 ESI 1 patients out of 11,025** (0.1%). This is a severely imbalanced problem, and the clinically most important group is also the rarest.

Two models were compared this week on this same split: **Logistic Regression** (a straightforward linear model, our current baseline) and **Random Forest** (an ensemble of 300 decision trees, the "more complex" model we tested as a potential upgrade).

---

## 2. Benchmark table

| Axis | Logistic Regression (baseline) | Random Forest (complex model) |
|---|---|---|
| Accuracy | **68.0%** | 52.8% |
| Precision (macro-average across all 5 ESI levels) | **0.57** | 0.37 |
| Recall (macro-average across all 5 ESI levels) | **0.48** | 0.43 |
| F1 score (macro-average) | **0.51** | 0.37 |
| **Recall for ESI 1 specifically** (the number we care about most) | **0.25** (4 of 16 caught) | **0.00** (0 of 16 caught) |
| Training time | Fast: a few seconds on this dataset | Slower: must build and combine 300 individual trees |
| Time to score one patient | Fast: a single calculation | Slower: every one of the 300 trees must be consulted and combined |
| Can a clinician's question ("why was this patient flagged?") be answered in under a minute? | **Yes.** Each input has one number attached to it, so its influence on the prediction can be read off directly | **Only partly.** The model can rank which inputs matter *on average* across all patients, but it cannot give a clean, one-line reason for *this specific* patient without additional tools we have not yet built (e.g. SHAP) |

Logistic Regression wins on every quantitative axis, not only on overall accuracy. Random Forest does not identify a single one of the 16 ESI 1 patients in the test set, despite being explicitly configured to weight rare classes more heavily, a result discussed further below.

---

## 3. Three arguments for keeping Logistic Regression

**1. It is simply the stronger model, on every measure we checked.** Accuracy, precision, recall, and F1 are all higher for Logistic Regression than for Random Forest. This is not a case of one model winning on one metric and losing on another; it wins across the board.

**2. It does better on the one number that matters clinically.** Logistic Regression correctly flags 1 in 4 of the truly critical (ESI 1) patients; Random Forest flags none of them. Neither figure is good enough to rely on alone (see Section 5), but between the two, Logistic Regression is meaningfully less likely to miss the sickest patients.

**3. It is cheap, fast, and easy to explain and audit.** A clinician or governance reviewer can look at the model's coefficients and see directly how each vital sign or demographic factor pushes a prediction up or down. Random Forest requires more compute to train and to score, and can only offer an *average* ranking of important features rather than a plain-language reason for one patient's specific result — a meaningful difference for anyone who has to explain or sign off on the model's use.

---

## 4. Three arguments against — the honest costs

**1. It is still not safe to use on its own.** Even our best model misses 3 out of every 4 truly critical patients (recall of 0.25). This is the central finding of this week's work, and it applies to Logistic Regression as much as to Random Forest: whichever model we keep, it cannot be trusted as a stand-alone filter for the sickest patients today.

**2. It is a linear model, and the true relationship between symptoms and urgency may not be linear.** Logistic Regression can only combine inputs in straightforward, additive ways. It is possible that some combination of vital signs and complaints, for example, a particular pattern across several near-normal readings, signals real danger in a way a linear model cannot capture. Random Forest's poor result this week does not rule out other, better-tuned non-linear approaches (for example, gradient boosting) succeeding where Random Forest did not.

**3. Its headline numbers can look more reassuring than they should.** A reported accuracy of 68% and a weighted F1 of 0.675 sound like solid performance to a non-specialist reader. Those figures are dominated by the 80% of patients in the middle two ESI categories, and say nothing about the 0.1% of patients in ESI 1. There is a real risk that quoting the headline numbers alone, without the ESI 1 recall figure alongside them, gives false reassurance to executives or governance reviewers who are not looking at the per-class breakdown.

---

## 5. Risks and unknowns

- **The ESI 1 sample is very small.** Only 16 patients in the test set are ESI 1, so a recall of 0.25 means exactly 4 patients were caught — one or two cases going the other way would change the percentage substantially. These figures should be read as indicative of a real problem, not as a statistically precise estimate.
- **We do not yet know why class weighting failed to help Random Forest.** The Random Forest was explicitly configured to give more weight to rare classes, which is the standard first fix for imbalance — yet it still caught zero ESI 1 patients. Whether this is because there are simply too few real examples for the trees to learn from, or because the hyperparameters need further adjustment, is an open question for next week.
- **We have not yet checked for fairness across patient subgroups.** The model uses age, gender, ethnicity, language, and other demographic fields as inputs. We have not yet tested whether errors, particularly missed ESI 1 patients, are spread evenly across these groups, or concentrated in one. A model that is broadly accurate but systematically misses one demographic group is a governance risk in its own right.
- **Neither model has been validated outside this dataset.** Both were trained and tested on data from the same source and time period. We do not know how either would perform on a different patient population, a different department, or at a different time of year.
- **Exact training and scoring times are still pending.** The comparison above is qualitative; timed code has been added to the modelling notebook but has not yet been run through to completion on the full dataset.

---

## 6. Recommendation

We recommend that **Logistic Regression remain the working baseline model** going into Week 8, and that **Random Forest, as tested this week, not replace it** — it underperformed on every axis we measured, including the clinically critical one.

We do **not** recommend deploying either model as a stand-alone triage decision-maker. Both should, for now, be treated purely as a decision-support flag that sits alongside — and never replaces — clinical judgement, with mandatory clinician review built into the workflow rather than left optional.

**What this recommendation does not solve.** Choosing Logistic Regression over this week's Random Forest does not fix the underlying problem that drove this whole exercise: neither model reliably identifies the most urgent patients. A recall of 0.25 for ESI 1 is not remotely close to a safe operating threshold. This choice also does not provide any assurance that the model treats patient subgroups fairly, and it does not remove the need for a human clinician to check every triage output before it is acted on. The next round of work should target ESI 1 recall directly, through resampling or cost-sensitive thresholds rather than accuracy-driven tuning, investigate why class weighting did not help Random Forest, and run a subgroup fairness check, before any properly-tuned non-linear model is considered as a genuine alternative to this baseline.
