# Feasibility Memo
**CariSurg MedTech Pathways | Week 5**
**Yale EMMLC Triage Extract → Baseline Triage Model**

## 1. Verdict
Proceed to a baseline triage model, with caveats. The ESI label and core vitals are usable after cleaning, but glucose, ethnicity, and insurance-status gaps, and the US-hospital sampling frame, must be named and mitigated before any Mercer deployment.

---

## 2. Dataset summary

- **Encounters:** 55,121 rows after dropping unlabelled ESI rows
- **Features:** ~225 columns per encounter — ~200 are sparse `cc_` chief-complaint flags (0/1), the rest are the structured demographic/vitals/outcome block profiled in Tutorial 2.
- **Target:** `esi`, a 5-level ordinal acuity code (1 = most urgent to 5 = least); most encounters sit mid-acuity (ESI 3), very few are ESI 1 
- **Demographics:** age range, gender split, race, ethnicity, insurance
- **Units flag:** triage temperature is recorded in **°F**, not °C

---

## 3. Top 3 concerns — named, with mitigation

| # | Concern | Evidence (from Tutorial 2 profiling) | Mitigation taken / planned |
|---|---|---|---|
| 1 | **Missingness, concentrated in a few columns** | `ethnicity` ~42% missing, `triage_glucose` ~37%, `insurance_status` ~16%, `triage_vital_o2` ~11% missing; `esi` itself ~1.8% missing | Vitals missingness handled via documented imputation; glucose's high gap likely reflects it not being drawn on every patient, a workflow gap rather than a random one |
| 2 | **Dtype inconsistencies masking numeric fields** | `triage_vital_rr` and `triage_vital_temp` load as text (~98% parseable) due to stray non-numeric entries (e.g. a "not recorded" placeholder, a stray unit suffix) | Coerced with `pd.to_numeric(..., errors="coerce")` in cleaning; flagged so a silent `.mean()`/model call doesn't quietly drop them |
| 3 | **Outcome leakage risk** | `disposition` (and any `previousdispo`-style field) is only known *after* the visit ends | Excluded from the candidate feature list on purpose, named explicitly so a reviewer doesn't have to find it themselves |

---

## 4. Top 3 reasons to proceed — evidence, not optimism

| # | Reason | Supporting evidence |
|---|---|---|
| 1 | **A real triage label, not a stand-in** | The target is learnable, not proxy-derived |
| 2 | **Vitals are usable after cleaning** | Once coerced/imputed, the vitals show separation across ESI levels in the clinically expected direction (Tutorial 2 vitals-vs-ESI correlation; Tutorial 4 box plots) |
| 3 | **A documented, reproducible pipeline** | Every missingness figure, dtype fix, and drop decision in this memo traces to a specific profiling table or cleaning-log row, auditor can follow it end-to-end |

---

## 5. Caveats

- **Units:** temperature is °F; any threshold or comparison must convert consistently.
- **Sparse chief-complaint flags:** many `cc_` columns are near-constant (<0.5% prevalence), low-value as individual model inputs; treat as a body-system aggregate rather than 200 separate features.
- **External validity untested:** this is a US academic-hospital sample; case-mix, demographics, and resourcing may not transfer to a Caribbean ED (Mercer).
- **Fairness-sensitive fields:** `race`, `ethnicity`, `insurance_status` carry real missingness *and* are potential bias vectors if used as model inputs.

