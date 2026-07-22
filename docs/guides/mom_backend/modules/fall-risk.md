# Fall Risk Assessment

Stores and serves fall-risk assessment data for patients, including anticholinergic/sedative/CNS burden scores, fall history, and IDT (interdisciplinary team) care planning output such as sequenced actions, monitoring schedules, and education points.

## Endpoints

| Method | Path | Purpose |
|---|---|---|
| POST | `/webapi/api/v1/fall-risk-assessment/` | Create or update a patient's fall risk assessment (upsert keyed on `patient_id`) |
| GET | `/webapi/api/v1/fall-risk-assessment/overview/{patient_id}` | Get risk level/score, current & projected burden scores, executive summary, fall history, IDT decisions |
| GET | `/webapi/api/v1/fall-risk-assessment/medication-plan/{patient_id}` | Get sequenced actions and top contributors (medications), enriched with checklist status |
| GET | `/webapi/api/v1/fall-risk-assessment/care-team/{patient_id}` | Get IDT decisions, monitoring schedule, and specialists required |
| GET | `/webapi/api/v1/fall-risk-assessment/education-context/{patient_id}` | Get education points and clinical context |
| GET | `/webapi/api/v1/fall-risk-assessment/{patient_id}` | Get the raw fall risk assessment record for a patient |
| GET | `/webapi/api/v1/fall-risk-assessment/{patient_id}/{run_id}` | Get the raw fall risk assessment record for a patient's specific analysis run |

## Data model

- **`fall_risk_assessment`** (`FallRiskAssessment`) — primary key `patient_id`. Written by the POST endpoint and read by the two `GET /{patient_id}...` endpoints.
  - Identity/meta: `run_id`, `report_date` (required, TIMESTAMPTZ), `patient_age`
  - Risk scoring: `fall_risk_level`, `fall_risk_score`, `current_acb_total`/`current_slm_total`/`current_cns_total`, `projected_acb_total`/`projected_slm_total`/`projected_cns_total`, `projected_risk_reduction`
  - Narrative: `executive_summary`, `source_recommendations_count`
  - JSONB fields: `patient_conditions`, `fall_history`, `top_contributors`, `sequenced_actions`, `monitoring_schedule`, `specialists_required`, `education_points`, `idt_decisions`, `clinical_context`

- **`fall_risk_idt_evaluations`** (`FallRiskIdtEvaluation`) — primary key `id`, keyed by `patient_profile_id` + `run_id`. Backs the `overview`, `medication-plan`, `care-team`, and `education-context` endpoints.
  - Scoring: same current/projected ACB/SLM/CNS totals, plus `fall_risk_level`, `fall_risk_score`
  - Counts: `top_contributors_count`, `sequenced_actions_count`, `specialists_required_count`, `education_points_count`, `idt_decisions_count`
  - `idt_report` (JSONB) — nested structure containing `executive_summary`, `fall_history`, `idt_decisions`, `sequenced_actions`, `top_contributors`, `monitoring_schedule`, `specialists_required`, `clinical_context`, `education_points`, `patient_age`
  - `created_at`, `overridden`, `overridden_at`

## Notes

- Two distinct data sources back this module: the POST/`GET {patient_id}` endpoints read/write `fall_risk_assessment` directly, while `overview`, `medication-plan`, `care-team`, and `education-context` instead resolve the patient's latest run via `get_latest_evaluation_run_id` and read from `fall_risk_idt_evaluations.idt_report` — these two tables are not automatically kept in sync by this code.
- All four `idt_report`-backed endpoints run their extracted fields through `clean_citation_obj`, filtering citations against a valid-ID set built from `individual_intelligence_reports.citation_map` for the same patient/run.
- The medication-plan endpoint enriches each `top_contributors` item with `status`, `change_reason`, and `change_notes` by looking up the latest `recommendation_checklist_mapping` row per `gcn_seqno`; lookup failures are swallowed and treated as "no status."
- On create/update, `report_date` is required and must be ISO-format; naive datetimes are assumed UTC and `Z` suffixes are normalized, otherwise a 400 is raised. Any `citation_map` in the request `data` is silently dropped since it isn't a column on `fall_risk_assessment`.
- The service maps database connection/timeout errors to 503 with a network-troubleshooting message, and all other unhandled exceptions to 500; missing records return 404 (with a structured `{error, message}` detail for the IDT-based endpoints).
