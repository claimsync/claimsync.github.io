# Quality Measures

This module computes a battery of 17 clinical quality/gap-closure measures per patient, evaluated against a specific evaluation run (`run_id`) and persisted per-tenant for value-based care and payer reporting. Each measure is implemented as an independent router → service → repository → schema stack, but all measures share one calculation shape (eligibility gating, then either a PDC/percentage calculation or a binary gap-open determination) and persist into a single wide `patient_quality_measures` table.

## Endpoints

| Measure | Method | Path | Purpose |
|---|---|---|---|
| C15 | POST | `/webapi/api/v1/quality_measures/c15/assess` | Fall Risk Management (docstring name). PACE patients 65+ with a documented fall, balance/gait problem, or HIGH/MODERATE fall risk score; gap open until a fall-prevention intervention is documented. |
| C16 | POST | `/webapi/api/v1/quality_measures/c16/assess` | Improving Bladder Control / MUI (docstring name; full expansion not given in code). |
| CBP | POST | `/webapi/api/v1/quality_measures/cbp/assess` | Controlling High Blood Pressure, CMS C14 (docstring name). |
| COB | POST | `/webapi/api/v1/quality_measures/cob/assess` | Concurrent Use of Opioids and Benzodiazepines (docstring name). Gap open when active opioid + active non-PRN benzo overlap concurrently, absent MAT/hospice exceptions. |
| GSD | POST | `/webapi/api/v1/quality_measures/gsd/assess` | Glycemic Status Assessment for Diabetes, CMS C12 (docstring name). |
| KED | POST | `/webapi/api/v1/quality_measures/ked/assess` | Kidney Health Evaluation for Patients with Diabetes, CMS C13 (docstring name). |
| MTM | POST | `/webapi/api/v1/quality_measures/mtm/assess` | MTM Program Completion Rate for CMR (docstring name). MTM-enrolled patients with ≥2 CMS qualifying chronic conditions and ≥8 active Part D meds; gap open until CMR completion + medication action plan documented. |
| OMW | POST | `/webapi/api/v1/quality_measures/omw/assess` | C10 Osteoporosis Management in Women (docstring name). |
| PCR | POST | `/webapi/api/v1/quality_measures/pcr/assess` | C18 — Plan All-Cause Readmissions (docstring name). No per-patient gap; a plan-level O/E outcome measure — endpoint only surfaces readmission-episode tracking and medication risk signals. |
| PDC-DIAB | POST | `/webapi/api/v1/quality_measures/pdc-diab/assess` | Proportion-of-days-covered adherence score for diabetes (non-insulin) medications; insulin presence and ESRD/dialysis are exclusion gates. Full measure name not specified in code. |
| PDC-RASA | POST | `/webapi/api/v1/quality_measures/pdc-rasa/assess` | PDC adherence score over a RASA medication class, excluding patients on sacubitril/valsartan and ESRD/dialysis. Full measure name not specified in code. |
| PDC-RASA (list) | GET | `/webapi/api/v1/quality_measures/pdc-rasa/` | Paginated list of each patient's latest results across all 17 measures in one response, with per-measure `*_eligible` filters and `patient_id`/`center_id` filters |
| PDC-Statin | POST | `/webapi/api/v1/quality_measures/pdc-statin/assess` | Statin adherence (PDC) — proportion-of-days-covered for statins; results also exposed via the PDC-RASA GET endpoint's `stat_eligible` filter |
| POLY-ACH | POST | `/webapi/api/v1/quality_measures/poly-ach/assess` | Polypharmacy: High-Risk Anticholinergic Medications (docstring name). Eligible if ≥2 anticholinergic (ACB) agents and ACB burden total ≥2; flags compounding with SAA (antipsychotic) and COB (benzo+opioid) risk. |
| SAA | POST | `/webapi/api/v1/quality_measures/saa/assess` | Antipsychotic Use in Persons with Dementia (docstring name). 65+, dementia diagnosis, active antipsychotic, no schizophrenia/bipolar/Huntington's/Tourette exclusion, not palliative; gap open with no documented exception. |
| SPC-E | POST | `/webapi/api/v1/quality_measures/spc-e/assess` | Statin Therapy for Patients with Cardiovascular Disease, CMS C19 (docstring name). Gap open unless eligible ASCVD patient has a HIGH/MODERATE-intensity statin ACTIVE. |
| SUPD | POST | `/webapi/api/v1/quality_measures/supd/assess` | Statin Use in Persons with Diabetes, CMS D12 (docstring name). Binary snapshot (no PDC%) — gap open if no qualifying ACTIVE statin for an eligible diabetic patient aged 40–75. |
| TRC | POST | `/webapi/api/v1/quality_measures/trc/assess` | C17 sub-rate 4 — Medication Reconciliation Post-Discharge (docstring name). |

All 17 measures are mounted through `webapi/quality_measures/router.py` (`prefix="/quality_measures"`) which includes `webapi/quality_measures/v1/router.py`, a flat router that attaches each measure's sub-router under its own kebab-case prefix (e.g. `poly_ach_router` → `/poly-ach`). Every measure exposes the identical `POST /{prefix}/assess` contract; PDC-RASA is the only sub-router with a second, `GET /`, endpoint.

## Data model

Nearly all measures persist into one shared table, `quality_measures.patient_quality_measures`, keyed by a unique `(patient_id, run_id)` pair. Rather than a normalized numerator/denominator schema, each measure gets its own nullable JSONB column (`pdc_rasa`, `pdc_diab`, `pdc_statins`, `supd`, `spc_e`, `cbp`, `gsd`, `ked`, `poly_ach`, `cob`, `saa`, `coa`, `mtm`, `c15`, `c16`, `omw`, `trc`, `pcr`) holding that measure's full result schema (eligibility, gap status, and enrichment fields) as a blob, plus a shared `assessed_at` timestamp. Note: a `coa` column exists in the model with no corresponding router among the 17 wired up — an unrouted/legacy column.

## Calculation pattern

- **Sequential eligibility gates**: every service runs an ordered chain of gates (missing DOB, age threshold, active-enrollment/deceased, palliative/hospice, condition- or medication-specific exclusions such as `insulin_excluded`, `esrd_or_dialysis`, `no_ach_medications`) and returns immediately with `eligible=False` + `ineligibility_reason` on the first failing gate — the code explicitly documents gate ordering as significant ("Gate order per spec").
- **Two distinct "gap" styles**: PDC-style measures compute a literal percentage — `days_covered` (union of merged, overlapping medication-fill date intervals) divided by `measurement_window` (IPSD-to-Dec-31 span) — with `gap_open = pdc_pct < 80.0`, plus `days_to_close`/`days_remaining` countdowns. Non-PDC measures (e.g. C15, POLY-ACH) instead compute a binary `gap_open` from whether a qualifying "closure" condition/intervention is documented (e.g., C15 checks three ranked closure sources — progress-notes NLP keyword scan, active Vitamin D prescription, FRID deprescribing pathway — falling through to `gap_open=True` only if none match).
- **IPSD (Index Prescription Start Date) derivation**: PDC measures derive an anchor date from the earliest qualifying medication start date in the measurement year, falling back to Jan 1 if none is found in-year — each fallback path is recorded as a `schema_gaps` entry.
- **Cross-measure compounding**: POLY-ACH recomputes `saa_compound` (antipsychotic overlap) and `cob_compound` (benzo+opioid overlap) inline rather than by cross-referencing SAA/COB results, using shared drug-classification helpers imported from its own repository module.
- **Persistence per assess() call**: all services persist the result at every return branch (including ineligible branches), so partial/ineligible outcomes are stored just like eligible ones.

## Notes

- Every `/assess` endpoint supports both a **targeted** run (`patient_id` + `run_id` in the body) and a **bulk** run (empty body → assess every patient's latest evaluation run); targeted-mode failures are swallowed into a generic `{"failed": 1}` counter with no error detail returned to the caller.
- Bulk assessment logs failures per-patient but continues iterating, so one patient's failure does not abort the batch — it only reduces the reported `assessed` count.
- Heavy use of self-documenting `schema_gaps`/`warnings` lists inside each result: services record, per computation, where the underlying EHR schema lacked a structured field and a heuristic/proxy was substituted (e.g., using `stop_date` as a proxy for `hold_date`, inferring `hold_reason` from free-text diagnoses, deriving `high_risk_agents` via a keyword scan when a structured fall-risk-assessment record is absent).
- The `pdc-rasa` `GET /` endpoint is the module's one cross-measure reporting surface: it reads the shared `patient_quality_measures` row and opportunistically parses each measure's JSONB column into its own result schema, individually swallowing per-column parse errors so one corrupt measure blob doesn't break the whole list response.
