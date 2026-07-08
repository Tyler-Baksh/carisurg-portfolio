# Feasibility Memo

**CariSurg MedTech Pathways | Week 5**
**Yale EMMLC Triage Extract → Baseline Triage Model**

---

## 1. Verdict

> **Proceed to a baseline triage model, with caveats.**

The dataset's triage-acuity label and core vital signs are complete and clinically usable once a small number of data-entry corrections are applied. It is suitable as the foundation for a baseline model in Week 6, provided the following concerns and caveats are respected and not glossed over: chief-complaint sparsity, modest individual vital-sign signal, and untested transferability to a Caribbean emergency department.

---

## 2. Dataset summary

- **55,121 emergency-department encounters**, each described by roughly **225 fields**. Only **25 of those fields** are structured demographic, vital-sign, or outcome measurements; the other **~200** are `cc_*` chief-complaint flags, a yes/no tick-box per possible presenting complaint (e.g. "chest pain: yes/no"), so almost all of them are "no" for any given patient.
- **Triage label:** the acuity level assigned at the front door — called **`Triage_Level`** in this memo, stored in the raw file as `esi` (the Emergency Severity Index). It runs from **1 (most urgent, e.g. resuscitation)** to **5 (least urgent)**. Every one of the 55,121 encounters carries a label. There are no unlabelled rows to drop in this extract.
- **Acuity mix** (see Figure 1): the great majority of patients sit in the middle of the scale, **49.0% are Level 3** and **32.5% are Level 2**, while the two ends of the scale are rare (**Level 1: 0.1%**, **Level 4: 16.1%**, **Level 5: 2.2%**). This mirrors real-world ED triage, where the most critical patients are a small minority; a model that only ever predicts "Level 3" would look accurate on paper while missing almost every genuinely urgent case, which is exactly the failure mode this project must guard against.
- **Vitals recorded at triage:** heart rate, systolic and diastolic blood pressure, respiratory rate, oxygen saturation, temperature, and blood glucose. All seven vitals have **0% missing values** in this extract, which is unusually complete for real clinical data and should be treated as a genuine strength rather than assumed without checking (see Concern 1 below for the one thing this clean figure does not tell us).
- **Units flag:** temperature is recorded in **°F** (normal ≈ 97–99.5°F), not °C. Any clinical threshold used later must be applied in the correct units.
- **Who is in the sample:** patients are adults (18+), average age in the mid-fifties with a wide spread (about two-thirds of patients are between roughly 35 and 75). The sample is 57.6% female; 53.4% White/Caucasian and 81.9% non-Hispanic by the categories recorded; 38.9% on Medicaid. These figures describe a single **US academic-hospital population** — see Caveat 4.

**Figure 1. Distribution of Triage_Level (ESI) across ED arrivals**

<img width="640" height="440" alt="Triage_Distribution" src="https://github.com/user-attachments/assets/bb0ef945-dd0f-494c-bc4c-1c8e9cff99f0" />

*Read as: most encounters are mid-acuity; the two ends of the scale (the most and least urgent patients) are both rare. Source: Week 5 data-exploration notebook, Section 11.*

---

## 3. Top 3 data-quality concerns

| # | Concern | Evidence (Week 5 profiling & outlier detection) | Mitigation taken / planned |
|---|---|---|---|
| 1 | **Two numeric columns were secretly categories, not measurements** | The acuity code (`esi`/`Triage_Level`, values 1–5) and the oxygen-device flag (`triage_vital_o2_device`, values 0–1) were both stored as plain decimal numbers. Left as-is, a script could quietly average them (e.g. report a "mean acuity of 2.9" or "half a nasal cannula"), which has no clinical meaning. | Both were re-typed as ordered/categorical fields in the cleaning step so they sort and display correctly and cannot be silently averaged. |
| 2 | **A handful of vitals carry more extreme/unusual values than the rest** | Statistical outlier checks (values far outside the typical range for that vital) flag **glucose** most often, **10.3%** of glucose readings are statistical outliers, and **25 readings (0.05%)** fall outside what is physiologically possible for a living patient. **Temperature** is next most affected (**6.5%** statistical outliers). Respiratory rate has a very small number (4 readings, 0.01%) that are physiologically impossible. | Physiologically-impossible readings (a data-entry error, not a sick patient) are candidates for recoding as missing rather than being fed to a model as real values; the larger share of "statistically unusual but plausible" readings (e.g. very high glucose) most likely reflect genuinely unwell patients and should be kept, not deleted, but reviewed once more before Week 6. |
| 3 | **The chief-complaint flags are extremely thin on their own** | ~200 individual yes/no complaint columns, each firing for only a small slice of patients. Individually, most carry too little signal to be useful model inputs, and several are close to always "no" for the whole sample. | Plan to group these into clinically meaningful bundles (e.g. "cardiac", "respiratory", "trauma") rather than feeding 200 near-empty columns into a model one at a time. |

---

## 4. Top 3 reasons to proceed

| # | Reason | Supporting evidence |
|---|---|---|
| 1 | **A real, complete triage label to learn from** | `Triage_Level` is a genuine clinical decision recorded at the time of arrival, not a stand-in or a proxy, and, as shown in Figure 1, every one of the 55,121 encounters has one. There is a real target to learn, and enough encounters at every level (even the rare Level 1 and Level 5 cases) to attempt a model. |
| 2 | **Vitals move in the clinically expected direction once cleaned** | After the dtype fix in Concern 1, every vital sign's relationship with acuity points the way a clinician would expect: older patients trend toward more urgent triage, and patients with a faster heart rate, faster breathing, or lower oxygen saturation also trend more urgent (see Figure 2). No vital moved in a clinically backwards direction. Individually these relationships are modest. Age is the strongest single signal, oxygen saturation second. This is why a model that weighs several vitals *together* is worth building, rather than relying on any one reading alone. |
| 3 | **A documented, reproducible pipeline an auditor can follow** | Every figure in this memo, the missingness count, the dtype fix, the outlier percentages, the acuity distribution, traces back to a specific, re-runnable cell in the Week 5 profiling notebook. A reviewer does not have to take our word for any number here; they can re-run the notebook against the same file and get the same tables. |

**Figure 2. Vital signs by Triage_Level**

<img width="1589" height="690" alt="VitalSignsbyTriageLevel" src="https://github.com/user-attachments/assets/2d941096-2321-4033-b1c6-f61d1dc0c864" />

*Read as: box plots show the typical range of each vital at every acuity level. Where the boxes shift consistently from Level 1 through Level 5, that vital is tracking acuity in the expected clinical direction. Source: Week 5 data-exploration notebook, Section 12.*

---

## 5. Caveats

- **Temperature is in °F, not °C.** Any fever threshold, alert rule, or comparison against a Caribbean chart must convert consistently, or it will silently misfire.
- **Correlation is association, not proof.** The vital-sign relationships in Figure 2 and Reason 2 show that vitals *tend to* move with acuity. They do not prove causation, and they are not a working model. Notably, systolic blood pressure showed almost no straight-line relationship with acuity (r ≈ 0.00) even though clinically both very low and very high blood pressure can signal danger — a plain correlation check cannot see that kind of U-shaped pattern, which is a reason to look more carefully at this vital in Week 6 rather than concluding it is unimportant.
- **Chief-complaint flags are sparse and numerous.** With ~200 near-empty yes/no columns, they should be grouped into clinical categories rather than used as 200 separate model inputs (see Concern 3).
- **Two fields must never be used as model inputs.** `disposition` (where the patient ended up) and `previousdispo` are only known *after* the visit is over. Using either as an input would let the model "see the answer" before triage happens, a mistake known as **information leakage**: training on data the model would not actually have at the moment it needs to make a prediction. Both are excluded from the candidate feature list on principle, not just for this exploration but for every later modelling step.
- **External validity for a Caribbean ED is untested.** This is a **US academic-hospital sample**: the case mix, demographics (see dataset summary), staffing, and available equipment may not match a Mercer emergency department. A model that performs well here is not yet shown to perform well there. This should be treated as an open question for Week 6 and beyond, not assumed away.

---

*Evidence for every figure in this memo is available in the Week 5 data-exploration notebook (dataset loading, missingness audit, dtype audit and corrections, outlier detection, distribution analysis, and correlation analysis against `Triage_Level`).*
