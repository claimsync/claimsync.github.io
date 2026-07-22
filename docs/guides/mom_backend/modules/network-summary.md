# Network Summary

This module exposes network- and center-level clinical analytics on top of aggregated `patient_statistics` data (ACB, fall risk, MAI, MAI urgency, and SLM metrics). It provides an overview dashboard across the tenant's patient population, a per-center breakdown, a filterable/sortable/paginated patient list, and a pin/unpin action for a user's patient list. Each read endpoint caches its JSON response for 120 seconds, keyed by tenant, auth token, and query parameters.

## Endpoints

| Method | Path | Purpose |
|---|---|---|
| GET | `/webapi/api/v1/patient_statistics_dashboard_simple` | Network-level overview: risk distributions (ACB, fall, MAI, MAI urgency, SLM, overall high-risk) and aggregate center metrics across all active patients |
| GET | `/webapi/api/v1/patient_statistics_dashboard_center_id` | Same risk distributions and metrics, grouped per center (optionally filtered to one `center_id`) |
| GET | `/webapi/api/v1/patient_statistics_simple` | Paginated, filterable, sortable list of patients with demographics, risk scores, and pin status |
| PUT | `/webapi/api/v1/pin_patient` | Pin or unpin a patient (`member_id`) for the requesting user |

## Data model

| Model | Table | Represents |
|---|---|---|
| `Center` | `centers` | Facility/center reference data (`center_id`, `center_name`) used to label statistics by center |
| `Participant` | (shared model) | Patient/member identity record (`member_id`, `participant_name`, `is_active`, `pinned_by`) joined to statistics rows |
| `PatientMedication` | `patient_medications` | Individual medication records per patient/evaluation run, used for medication-name search |
| `PatientProfile` | `patient_profiles` | Patient demographic/context JSON (height, weight, diagnoses) sourced per evaluation run |
| `PatientStatistics` | `patient_statistics` | Core aggregated risk record per patient/run: `overall_risk_level`, `acb_metrics`, `fall_risk_metrics`, `mai_metrics`, `slm_metrics`, `patient_summary` (age, diagnoses count, medication count) |

## Notes

- All three GET/PUT endpoints wrap repository/service calls in broad `try/except` blocks and return a generic 500 with `{"success": false, "error": "Internal server error", "message": str(e)}` on any exception — no distinction between DB errors and business-logic errors.
- Each row is scoped to the patient's latest completed evaluation run via a `LATERAL` subquery joined on `EvaluationRun.status == "completed"`; a `X-User-Role-ID` header is checked against `ROLE_PERMISSIONS["validate_evaluation_run"]` to decide whether to additionally require `EvaluationRun.validated == True` (regular users are filtered to validated runs, privileged/reviewer roles are not).
- `patient_statistics_simple` uses a CTE with `DISTINCT ON (patient_profile_id)` to dedupe to one row per patient (most recent run), then a separate count CTE for total records; sorting on `medication_count`, `diagnoses`, and `mai_urgency` only applies if the underlying text value matches a numeric regex, else it sorts as `0`; `fall_risk` sorts via a hardcoded severity ranking (low=1, moderate/medium=2, high=3, severe/critical=4).
- When `current_user_id` is supplied to `patient_statistics_simple`, pinned patients (via `array_position` on `Participant.pinned_by`) are always sorted first, ahead of the requested sort order.
- `pin_patient` requires a non-empty `X-User-ID` header (401 if missing) and non-empty `patient_id` (400 if missing); pinning only inserts the user id if not already present while unpinning always attempts removal, and only active participants (`is_active = True`) are affected.
- Height/weight values in the patient list are parsed from raw stored value+unit pairs (`feet/inches`, `cm`, `inches` for height; `lbs`, `kg` for weight) into normalized `height_cm`/`weight_kg` plus structured `height`/`weight` objects; unparseable or unit-less values silently become `None`.
- When no patients match, `patient_statistics_dashboard_simple` returns a fixed zero-value response shape (all distributions empty, all scores 0) rather than omitting fields.
