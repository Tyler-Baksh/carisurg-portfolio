# Top-10 Feature Shortlist

**CariSurg MedTech Pathways | Week 5**
**Baseline Triage Model - Candidate Feature Ranking**

---

Ranked by combined strength of single-variable signal (Pearson correlation with `Triage_Level`, from Section 13 of the Week 5 data exploration notebook) and clinical plausibility as a driver of acuity.

| Rank | Feature | Justification |
|---|---|---|
| 1 | **Age** | Age has the strongest single-variable correlation with `Triage_Level` in the dataset (r = −0.237, i.e. older patients trend toward more urgent triage), which matches clinical intuition that older patients have less physiological reserve and more comorbidities to decompensate from an acute presentation. |
| 2 | **Oxygen saturation (`triage_vital_o2`)** | The second-strongest correlate (r = 0.178, higher acuity associated with lower saturation) and clinically one of the most direct proxies for respiratory or circulatory failure, the kind of derangement ESI triage is specifically designed to catch early. |
| 3 | **Respiratory rate (`triage_vital_rr`)** | Tied for third-strongest correlation (r = −0.095) and clinically one of the earliest vitals to shift in sepsis, respiratory distress, or shock, often abnormal before blood pressure drops. |
| 4 | **Heart rate (`triage_vital_hr`)** | Also r = −0.095, and clinically a fast, cheap, universally-available signal of compensatory stress (pain, hypovolemia, fever, hypoxia) that a triage nurse checks within the first minute. |
| 5 | **Blood glucose (`triage_glucose`)** | r = −0.078, and clinically both very low glucose (risk of loss of consciousness/seizure) and very high glucose (DKA/HHS) are acute triage red flags, so this feature carries real signal despite also being the vital with the most data-quality noise (10.3% statistical outliers per the feasibility memo, meaning it needs the cleaning step applied before use). |
| 6 | **Diastolic blood pressure (`triage_vital_dbp`)** | Weak but non-trivial correlation (r = 0.046) and clinically a component of perfusion pressure — a very low DBP can signal impending hemodynamic collapse even when systolic pressure is still compensating. |
| 7 | **Oxygen delivery device (`triage_vital_o2_device`)** | Not part of the correlation table (it's categorical, not continuous) but clinically meaningful on its own: a patient already on supplemental oxygen at triage (nasal cannula, non-rebreather) is by definition sicker than one on room air, so this flag adds information the raw saturation number alone can't fully capture. |
| 8 | **Temperature (`triage_vital_temp`)** | Correlation is close to zero (r = −0.022), but clinically fever or hypothermia are core sepsis-screening criteria, and the memo notes temperature also has a substantial outlier rate (6.5%) that likely reflects real febrile illness rather than just noise, so it belongs in a multivariate model even though it's weak alone. |
| 9 | **Systolic blood pressure (`triage_vital_sbp`)** | Correlation is essentially zero (r ≈ 0.001), but the memo explicitly flags this as a case where a straight-line correlation is the wrong tool: both very low (shock) and very high (hypertensive emergency) SBP are dangerous, a U-shaped relationship a linear correlation can't detect, so SBP is included on clinical grounds with the caveat that it may need to be modeled non-linearly (e.g. distance from a normal midpoint) rather than dropped. |
| 10 | **Chief-complaint bundle: "cardiac" or "respiratory" grouped flag** | Individually the ~200 `cc_*` columns are too sparse to use one at a time, but clinically the presenting complaint is one of the first things a triage nurse hears, so grouping related flags (e.g. `cc_chestpain`, `cc_shortnessofbreath`, `cc_palpitations` → one "cardiac/respiratory" bundle) turns a near-empty column into a plausible categorical feature that reflects real triage reasoning rather than noise. |

---

### Note on exclusions

`disposition` and `previousdispo` are deliberately left off this list on principle. Both are flagged as post-visit fields that would leak the outcome into a model that's supposed to predict acuity *at* arrival (information leakage).
