# Medication Optimization

This module covers the medication optimization workflow: ingesting/refreshing AI-generated triage and recommendation data for a patient's evaluation run, tracking per-medication recommendation status changes and their approval history, managing a clinical schedule checklist derived from that triage data, and generating/persisting provider-communication PDF snapshots for individual recommendations. It has no module-level path prefix — each sub-router defines its own path directly under `/webapi/api/v1/`.

## Endpoints

**Triage / recommendations** (`webapi/optimization/v1/routers/triage.py`, no prefix)

| Method | Path | Purpose |
|---|---|---|
| POST | `/webapi/api/v1/medication-optimization-triage/` | Create/upsert triage data for a patient run; distributes payload across the 4 optimization tables (triage, recommendations, intelligence reports, checklist) in one transaction |
| GET | `/webapi/api/v1/medication-optimization-triage/{patient_id}` | Get triage data for a patient's latest evaluation run (reads summary/strategic_decision from `recommendation_evaluations`, clinical_schedule from the triage table) |
| GET | `/webapi/api/v1/medication-optimization-triage/{patient_id}/{run_id}` | Get triage data for a specific patient + run ID |
| GET | `/webapi/api/v1/medication-optimization-recommendations/{patient_id}` | Get all recommendations for the patient's latest completed run, enriched with status, clinical intelligence/guideline findings, cleaned citations, PRN status, and clinical schedule |
| POST | `/webapi/api/v1/refresh/{patient_id}/{run_id}` | Re-populate all 4 optimization tables from the source `recommendation_evaluations` row for that patient+run (delete + re-insert) |

**Medication status** (`webapi/optimization/v1/routers/medication_status.py`, prefix `/medications`)

| Method | Path | Purpose |
|---|---|---|
| POST | `/webapi/api/v1/medications/status` | Update a medication's status (pending/approved/declined/needs_discussion) with transition validation; inserts a new status record per affected checklist item |
| GET | `/webapi/api/v1/medications/status/history` | Get full status-change history for a medication, optionally filtered by run_id |
| GET | `/webapi/api/v1/medications/{patient_id}/recommendation-kpi` | Get counts of recommendations by status (total/approved/pending/declined/needs_discussion) for a patient's run (run_id required) |

**Clinical schedule checklist** (`webapi/optimization/v1/routers/clinical_schedule.py`, prefix `/clinical-schedule`)

| Method | Path | Purpose |
|---|---|---|
| GET | `/webapi/api/v1/clinical-schedule/{patient_id}/checklist` | Get checklist items for a patient (filterable by period/primary_key/secondary_key/tertiary_key), scoped to the latest evaluation run, with citations cleaned and section-header rows excluded |
| GET | `/webapi/api/v1/clinical-schedule/{patient_id}/checklist/kpi` | Get checklist completion KPI (total items, completed items, overall progress %) for a patient's run (run_id required) |
| PATCH | `/webapi/api/v1/clinical-schedule/checklist/{item_id}` | Update check status (is_check + checked_by) of a single checklist item |
| PATCH | `/webapi/api/v1/clinical-schedule/checklist/bulk` | Bulk update check status of multiple checklist items |

**Medication optimization PDF** (`webapi/optimization/v1/routers/medication_optimization_pdf.py`, prefix `/medication-optimization-pdf`)

| Method | Path | Purpose |
|---|---|---|
| GET | `/webapi/api/v1/medication-optimization-pdf/{recommendation_id}` | Get PDF/provider-communication card data for a recommendation; returns the latest saved snapshot if present, otherwise generates it from the source recommendation and saves it as a system snapshot |
| POST | `/webapi/api/v1/medication-optimization-pdf/{recommendation_id}` | Save a user-edited PDF data snapshot (append-only; becomes the new "latest" returned by subsequent GETs) |

## Data model

- **MedicationOptimizationTriage** (`medication_optimization_triage`, composite PK `patient_id` + `run_id`) — stores `summary`, `strategic_decision` (JSONB), `burden_reduction_projection` (JSONB), and `clinical_schedule` (JSONB) per patient run.
- **MedicationOptimizationRecommendation** (`medication_optimization_recommendations`, PK `id`) — one row per recommendation, keyed by `patientid` + `run_id`, storing `recommendation` and `contra_view` as JSONB.
- **IndividualIntelligenceReport** (`individual_intelligence_reports`, PK `id`) — one row per medication per patient run, storing `clinical_intelligence` (JSONB) and `citation_map` (JSONB list) plus `gcn_seqno`.
- **ClinicalScheduleChecklist** (`medication_optimization.clinical_schedule_checklist`, PK `id`) — flattened checklist items derived from a triage's `clinical_schedule`, hierarchically keyed by `period` / `primary_key` / `secondary_key` / `tertiary_key` / `sequence`, with `data` (JSONB), `is_check`, `checked_at`, `checked_by`.
- **RecommendationChecklistMapping** (`medication_optimization.recommendation_checklist_mapping`, PK `id`) — append-only status-history table linking a `recommendation_id` + `checklist_id` to a `status` (pending/approved/declined/needs_discussion), `previous_status`, change reason/notes, and who changed it (`changed_by_user_id`/`changed_by_user_name`); current status is derived by taking the most recent row per (patient_id, gcn_seqno).

## Notes

- The triage create/refresh flow fans one payload out across 4 tables atomically: `medication_optimization_triage`, `medication_optimization_recommendations`, `individual_intelligence_reports`, and `clinical_schedule_checklist`; the refresh endpoint (`POST /refresh/{patient_id}/{run_id}`) re-derives that same payload from `recommendation_evaluations` and deletes+re-inserts across all 4 tables.
- `get_triage_by_patient`/`get_triage_by_patient_and_run` intentionally read `summary`/`strategic_decision`/`burden_reduction_projection` from `recommendation_evaluations` (not the triage table) to match legacy middleware behavior, while `clinical_schedule` still comes from the triage table itself.
- Checklist retrieval filters out synthetic "section header" rows (items whose `display_suggestion`/`clinical_context` is wrapped in `---`) and scopes results to the patient's latest evaluation run via `get_latest_evaluation_run_id`.
- Citation cleaning (`clean_citation_obj`) is applied in three separate places (checklist GET, recommendations GET, PDF GET) using a valid-citation-ID set derived from `individual_intelligence_reports.citation_map` for that patient/run/medication.
- The PDF service caches a generated LLM summary back onto the recommendation row itself (`recommendation._pdf_summary` JSONB patch) so it isn't regenerated on every call, and separately persists full response snapshots (system- or user-generated) to `medication_optimization.pdf_snapshots`, gracefully degrading (logs a warning, continues) if that table doesn't exist.
- `POST /medications/status` requires `run_id` and validates the status transition before inserting a new row per affected checklist item — status history is append-only, never updated in place, mirroring the `RecommendationChecklistMapping` design.
- `GET /medication-optimization-recommendations/{patient_id}` and the checklist/KPI endpoints all depend on `get_latest_evaluation_run_id`/an explicit `run_id` query param rather than always using the newest triage row, so callers must be aware results are scoped to a specific run.
