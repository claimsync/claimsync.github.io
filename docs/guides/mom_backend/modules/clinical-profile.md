# Clinical Profile

The Clinical Profile module aggregates patient-facing clinical data: a per-patient evaluation/context timeline, EHR "wiki" snapshots tied to evaluation runs, a searchable/paginated patient roster, and a batch statistics endpoint that stitches together falls-risk, MAI, SLM, ACB, DDI, and Hub-analysis data from other modules. All endpoints run through a per-request tenant database session and can be scoped to only "validated" evaluation data via a role-based header check.

## Endpoints

| Method | Path | Purpose |
|---|---|---|
| GET | `/webapi/api/v1/patient-profiles/{patient_profile_id}/context` | Context — return all evaluation runs for a patient with the patient_context and personal details snapshot active at each run's creation time |
| GET | `/webapi/api/v1/ehr-wiki` | EHR wiki — return EHR wiki records for a patient's latest completed evaluation run (`patient_profile_id` query param) |
| GET | `/webapi/api/v1/patients` | Patients — paginated, searchable, sortable list of active participants |
| POST | `/webapi/api/v1/stats/patients/summary` | Patient stats — batch summary of falls risk, MAI, SLM, ACB, DDI, and demographic stats for a list of patient_profile_ids |

## Data model

- **`EvaluationRun`** (shared model) — evaluation run records; queried for `status == "completed"` and optionally `validated == True`; drives which run's data is shown in both context and EHR-wiki endpoints.
- **`participants`** (raw SQL, no ORM model in this module) — demographic/enrollment fields (`member_id`, `participant_name`, `team`, `center_id`, `date_of_birth`, `age`, `sex`, `living_situation`, `reimbursement_type`, `acuity_score`, `acuity_level`, `enrollment_date`, `disenrollment_date`/`reason`); source for the patients list and for "personal details" in the context response.
- **`patient_profiles`** (raw SQL, no ORM model in this module) — holds `patient_context` JSON per profile snapshot, matched to a run by `created_at <= run_created_at`.
- **`EhrWiki`** (table `public.ehr_wiki`) — `patient_profile_id`, `run_id`, `result` (JSONB), `created_at`; written elsewhere, read-only here.
- **`FallsRiskEvaluation`** (table `falls_risk_evaluations`) — total/component fall-risk scores, `risk_level`, `components`, `risk_messages`, override flags; latest row per patient feeds the stats endpoint.
- Cross-module models read (not owned) by the stats sub-area: `ACBEvaluation` ([acb](acb.md)), `MaiEvaluation` ([mai](mai.md)), `SLMEvaluation` ([slm](slm.md)), `HubAnalysis` ([hub](hub.md) — `mean_mai_urgency` pulled via JSONB path), `Participant`/`PatientProfile` ([network_summary](network-summary.md)), `PatientMedication` ([recommendations](recommendations.md), used for DDI medication lists).

## Notes

- `get_validated_filter` (in `webapi/core/dependencies.py`) inspects the `X-User-Role-ID` header against `ROLE_PERMISSIONS['validate_evaluation_run']`; privileged roles see all evaluation runs, everyone else is implicitly restricted to `validated = true` rows in both the context and EHR-wiki endpoints.
- `get_tenant_db` resolves a tenant-scoped SQLAlchemy engine/session from `request.state.tenant_db_name` (set by tenant middleware), so every endpoint in this module is tenant-isolated at the DB-session level.
- The EHR-wiki endpoint always resolves to the single most recent completed (optionally validated) evaluation run, not the full run history, unlike the context endpoint which returns all runs.
- `list_patients` accepts an `org_id` query parameter and echoes it back in the response's `filters` block, but the underlying query never actually filters by `org_id` — only `center_id` and `search` (participant_name `ILIKE`) affect the SQL.
- `sort_by` for the patients list is validated against a hardcoded allow-list (`member_id`, `participant_name`, `team`, `age`, `enrollment_date`, `acuity_score`) before being interpolated into raw SQL, since it can't be parameterized normally.
- Context responses are enriched in the service layer, not stored: height/weight are converted into `cm`/`inches`/`feet` and `kg`/`lbs` structures, `concurrent_medications` is filtered to `ACTIVE`/`PLANNED` status, and `total_daily_doses` is computed from medication frequency data.
- The batch stats endpoint (`POST /stats/patients/summary`) caps processing at `max_patients` (client-supplied, itself capped at 100) and continues past per-patient failures — errors are collected per patient rather than failing the whole request — and it makes a live call into `DDIService.get_interaction_matrix` per patient for drug-drug interaction scoring.
