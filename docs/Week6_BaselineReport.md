# Week 6 Baseline Report: Predicting ED Triage Urgency (ESI)

*CariSurg MedTech Pathways · Week 6 · Tutorial 4*

---

## 1. Dataset Recap

The models are trained on `triage_cleaned_v1.csv`, the cleaned output of the Week 5 data exploration notebook: **55,121 patients** and **225 columns**.

The prediction target is **ESI** (Emergency Severity Index), a 5-level scale where **ESI 1 is the most urgent** (needs immediate life-saving intervention) and **ESI 5 is the least urgent** (routine, can safely wait).

Feature groups used as model input:
- **Vitals at the front door**: heart rate, systolic/diastolic BP, respiratory rate, O2 saturation, temperature, glucose
- **Demographics**: age, gender, ethnicity, race, language, religion, marital status, employment status, insurance status
- **Chief complaint flags**: one-hot indicators for the presenting complaint (`cc_*`)

Two groups were deliberately **excluded** from the features: administrative/arrival fields (department, arrival mode, arrival month/day/hour) and outcome fields (`disposition`, `previousdispo`). The latter are only known *after* triage has already happened, so including them would leak the answer into the model.

After one-hot encoding the categorical demographics, 217 raw features became **262 model inputs**. The data was split 80/20 (stratified by ESI, `random_state=42`) into **44,096 training** and **11,025 test** patients.

The test-set class distribution (which mirrors the full dataset) is the single most important fact in this report:

| ESI level | Meaning | Test-set count | Share of test set |
|---|---|---|---|
| 1 | Immediately life-threatening | 16 | 0.1% |
| 2 | High risk / emergent | 3,585 | 32.5% |
| 3 | Urgent | 5,402 | 49.0% |
| 4 | Less urgent | 1,779 | 16.1% |
| 5 | Non-urgent | 243 | 2.2% |

This is a **severely imbalanced target**. The clinically most important class (ESI 1) is also the rarest by a wide margin, at roughly 1 in every 700 patients.

---

## 2. Model Descriptions

Three models were trained on the same train/test split:

- **Dummy Classifier** (`strategy="stratified"`): predicts ESI labels at random, in proportion to how often each class appears in training. This is the floor every real model must beat.
- **Logistic Regression**: a linear model on standardized features (`StandardScaler`), `max_iter=1000`.
- **Decision Tree**: `max_depth=15`. This depth was chosen after comparing 5, 10, 15, and 20: test accuracy rose substantially up to depth 15 (54.74% → 59.35%) but only marginally beyond it (59.45% at depth 20). The extra 0.10 percentage points at depth 20 didn't justify the added complexity and overfitting risk, so depth 15 was kept.

---

## 3. Benchmark Table

| Model | Accuracy | Macro F1 | Weighted F1 | Recall (ESI 1) |
|---|---|---|---|---|
| Dummy (baseline) | 37.5% | 0.204 | 0.375 | 0.00 |
| Logistic Regression | 68.0% | 0.509 | 0.675 | 0.25 |
| Decision Tree | 59.4% | 0.314 | 0.542 | 0.12 |

Both real models beat the dummy on every column. Logistic Regression is the stronger model overall.

---

## 4. Macro F1 vs Weighted F1 — why they disagree here

Both metrics average the per-class F1 scores, but they weight the classes differently, and on an imbalanced target that difference is not cosmetic:

- **Macro F1** gives every class equal weight, regardless of how many patients are in it. ESI 1 (16 patients) counts exactly as much as ESI 3 (5,402 patients).
- **Weighted F1** gives each class weight in proportion to its support (how many patients are actually in it). Common classes dominate the average; rare classes barely move it.

For Logistic Regression, this produces a **0.166-point gap** between Macro F1 (0.509) and Weighted F1 (0.675). The model does well on the common classes (ESI 2 and 3 F1-scores of 0.67 and 0.72), and because those classes make up over 80% of the test set, the weighted average mostly reflects that. But the model does poorly on the rare classes — ESI 1 (F1 = 0.32) and ESI 5 (F1 = 0.22). Because macro F1 treats those rare classes as equally important, it drags the macro score down to reveal that weakness.

The practical consequence: **weighted F1 (or accuracy) can look reassuring while the model is quietly failing the rarest, highest-stakes patients.** For a triage model, that is exactly the failure mode we cannot afford to hide behind an aggregate number.

---

## 5. Primary Metric Justification

**Chosen primary metric: Recall for ESI 1.**

In an emergency department, ESI 1 identifies the patients who need immediate, life-saving intervention where minutes matter. Missing one of these patients (a false negative) is the single worst error a triage model can make: a patient who needed to be seen immediately is instead routed into a slower queue, and the delay itself can cause irreversible harm or death. A false alarm on a lower-acuity patient, by contrast, costs staff time but is recoverable. Overall accuracy is a poor guide here precisely because of the imbalance documented above: ESI 1 patients are only 0.1% of the test set, so a model can score 68% accuracy, as Logistic Regression does, while still missing 3 out of every 4 of the sickest patients (recall = 0.25). Accuracy, and even weighted F1, are dominated by how well the model handles the common ESI 2/3 middle of the distribution, and can stay high even as ESI 1 recall collapses toward zero, as the Dummy Classifier illustrates (68%-range accuracy is not reached by dummy, but its ESI 1 recall is exactly 0.00 despite a "reasonable-looking" 37.5% overall accuracy). 

---

## 6. Failure Mode Reflection

**Which class does the model miss most?** ESI 1, the most urgent class, is missed most often across all three models. Logistic Regression, our best-performing model overall, still only recalls 0.25 (4 of 16) of the ESI 1 patients in the test set; the Decision Tree recalls 0.12 (roughly 2 of 16); the Dummy Classifier recalls 0.00.

**Why?** Two compounding reasons:
- *Extreme rarity*: ESI 1 makes up only 0.1% of patients, so the model sees very few training examples of what a true emergency looks like in this feature space, and has little signal to learn from.
- *Subtle presentation*: some ESI 1 patients may not present with dramatically abnormal front-door vitals. A subtle presentation can look, on paper, similar to a lower-acuity patient, and both models tend to default toward the common classes (ESI 2/3) when uncertain.

**What would this mean for a patient?** A missed ESI 1 patient is a patient with an immediately life-threatening condition who is instead routed into a queue meant for less urgent cases. In practice, that means a delay in the interventions that matter most in the first minutes at exactly the point where delay is most dangerous. With a test-set sample of only 16 ESI 1 patients, these recall numbers should be read as directionally consistent rather than statistically precise, but the consistent pattern across all three models (dummy, logistic regression, and decision tree) all missing this class most is itself the signal: **none of today's baseline models are safe to deploy as a stand-alone filter for the sickest patients.**

**Next step:** Week 7's tuning should target ESI 1 recall specifically, e.g. class weighting or resampling to counter the rarity, and a closer look at which specific vitals/complaint combinations are being confused with lower-acuity classes, rather than optimizing for overall accuracy or weighted F1, both of which would reward the model for continuing to ignore this class.

