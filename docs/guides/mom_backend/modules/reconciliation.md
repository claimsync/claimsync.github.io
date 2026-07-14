# Medication Reconciliation

This module compares a patient's current vs. previous medication list and classifies each medication as NEW, MODIFIED, DISCONTINUED, or UNCHANGED. It exposes three read-only "reports" endpoints that surface pre-computed reconciliation data produced by the [EHR pipeline's monthly report flow](../../ehr_pipeline/monthly_report/monthly_report_flow.md), plus one "live reconcile" endpoint that computes reconciliation on demand by comparing a patient's two most recent completed evaluation runs through a reconciliation engine (routed to an external `jerimed_service` first, falling back to an in-process `core_business_logic` engine).

## Endpoints

| Method | Path | Purpose |
|---|---|---|
| GET | `/webapi/api/v1/reconciliation/reports/monthly-medication-report` | Paginated, filterable, sortable list of all patients' latest-month reconciliation summary (counts only, no full change report), joined with monthly score deltas (MED, ACB, SLM, MAI, falls, hospitalizations) |
| GET | `/webapi/api/v1/reconciliation/reports/monthly-medication-report/months` | Distinct months available in `monthly_report_scores`, optionally filtered by center, sorted descending |
| GET | `/webapi/api/v1/reconciliation/reports/monthly-medication-risk/patients/{patient_id}/adherence-detail` | Per-month reconciliation detail for a single patient, including full `reconciliation_report`, score history ("periods"), falls, and hospitalizations |
| GET | `/webapi/api/v1/reconciliation/reconciliation/{patient_id}/reconcile` | Runs live reconciliation between the patient's latest two completed evaluation runs and returns categorized changes (new/modified/discontinued/unchanged) |

## Data model

No ORM `models/` folder is used here; the service reads/joins existing tables written by the batch monthly-report pipeline (and, for live reconcile, the raw evaluation data):

- **monthly_report_scores** — one row per patient per month: `current_*`/`previous_*` pairs for med count, ACB, SLM, MAI, falls, hospitalizations, fall risk score, MAI urgency, plus risk-level strings and `current_run_id`/`previous_run_id`/run dates. Joined via `medication_reconciliation_results_id` to reconciliation results, and via `patient_profile_id` to `participants`.
- **medication_reconciliation_results** — `new_count`, `modified_count`, `discontinued_count`, `unchanged_count`, `total_medications_current`, `total_medications_previous`, `is_baseline`, and a JSON `reconciliation_report` blob whose `changes` key holds `new`/`modified`/`discontinued`/`unchanged` lists of medication change items (name, strength, frequency, route, GCN seqno, PRN, TDD, ACB/SLM scores, clinical significance).
- **participants** — patient demographics (`participant_name`, `age`, `sex`, `center_id`), joined on `member_id = patient_profile_id` and `is_active = TRUE`.
- **patient_medications** — raw per-run medication rows (name, GCN seqno, route, dose, frequency, `is_prn`, start/stop dates) used both to backfill PRN flags into stored reports and as direct input to the live reconciliation engine.
- **evaluation_runs** — used only by the live-reconcile endpoint to find the patient's latest 2 rows with `status = 'completed'`.
- **patient_falls** / **patient_hospitalizations** — queried in the patient-detail endpoint for fall/hospitalization event history (from a fixed cutoff of 2025-06-01).

## Notes

- Reports vs. live reconcile: the three `/reports/...` endpoints only read pre-computed results from `monthly_report_scores` + `medication_reconciliation_results` (produced by the batch pipeline) — they never invoke the reconciliation engine. Only `/{patient_id}/reconcile` computes reconciliation on the fly.
- Nested prefix quirk: `webapi/reconciliation/router.py` mounts the "no-prefix" `reconcile_router` with `prefix="/reconciliation"`, and the whole module is itself mounted at `/reconciliation` one level up, producing a doubled path segment: `.../reconciliation/reconciliation/{patient_id}/reconcile`.
- PRN backfill: the batch pipeline never populates `is_prn`, so stored `reconciliation_report` blobs always have `current_prn`/`previous_prn` as false; `_enrich_reports_with_prn_status` patches these in from live `patient_medications` rows scoped to the exact run IDs referenced by each report before returning patient detail.
- Live reconcile requires at least 2 completed `evaluation_runs` for the patient — raises 404 if none exist and 400 if only one exists ("Need at least 2 runs to compare"). It first tries `post_jerimed(...)` to an external `jerimed_service` (if `JERIMED_EXTERNAL_SERVICE_URL` is set) and falls back to the in-process `_call_reconciliation_engine` on any failure.
- `get_patient_detail` returns `None` (not an exception) when no rows are found for a patient; the router translates that into a 404 with a `"Not found"` payload. `get_report` and `get_available_months` catch all other exceptions and return a generic 500.
- `sort_by`/`sort_order` on the list report accept comma-separated multi-field values; any field not in a fixed whitelist (`patient_name`, `patient_id`, `med`, `med_delta`, `acb`, `acb_delta`, `slm`, `slm_delta`, `mai`, `mai_delta`, `falls`, `falls_delta`, `hosp`, `hosp_delta`, `new_count`, `modified_count`, `discontinued_count`, `unchanged_count`) is silently dropped, defaulting to `patient_name`/`asc`.
- Month filters accept `MM/YYYY`, `YYYY/MM`, or `YYYY-MM` and are normalized to `MM/YYYY` internally via `_normalize_month`.
