# Opioid

This module retrieves opioid risk evaluations for a patient, returning the total daily morphine milligram equivalent (MME), a derived risk level, and a detailed opioid report (enriched with PRN medication status) computed by an upstream evaluation run.

## Endpoints

| Method | Path | Purpose |
|---|---|---|
| GET | `/webapi/api/v1/opioid/evaluations/{patient_profile_id}` | Get the latest (or, with `all=true`, every) opioid evaluation for a patient profile |

Query parameters: `all` (bool, default `false`) — return all evaluations instead of only the most recent one; also depends on `get_validated_filter` to control whether only validated-run evaluations are returned.

## Data model

`OpioidEvaluation` (table: `opioid_evaluations`)

| Field | Type | Notes |
|---|---|---|
| `id` | int | primary key |
| `patient_profile_id` | str(100) | indexed |
| `run_id` | UUID | links to the evaluation run that produced this evaluation |
| `opioid_report` | JSON | detailed report payload (e.g. `medication_details`, may include `risk_level`, `total_daily_mme`) |
| `total_daily_mme` | Numeric | total daily morphine milligram equivalent |
| `risk_level` | str | stored risk classification |
| `medications_with_mme` | int | count of medications contributing to MME |
| `alerts_count` | int | count of alerts raised |
| `overridden` | bool | flags a superseded/invalidated evaluation |
| `overridden_at` | datetime | when the evaluation was overridden |
| `created_at` | datetime | server-default `CURRENT_TIMESTAMP` |

## Notes

- Evaluations are only returned if their `run_id` belongs to an `EvaluationRun` with `status == "completed"`, and (unless the validated filter is disabled) `validated == True`; rows with `overridden = True` are always excluded.
- Risk level is derived by `_determine_risk_level`: uses `risk_level` from the report JSON if present, otherwise buckets `total_daily_mme` as None (0), Low (<50), Moderate (<90), High (>=90); falls back to "None"/"Unknown" if no MME or medication details exist.
- The opioid report's `medication_details` are enriched with an `is_prn` flag looked up from `PatientMedication` by GCN sequence number and run_id; if the `PatientMedication` model can't be imported, PRN enrichment is silently disabled (logs a warning) and `is_prn` lookups return empty.
- If no evaluations are found for the patient, the service raises a 404 with a structured `{success, error, message}` body; unexpected repository/service errors are wrapped into a 500 with the same structured body shape.
- With `all=true`, results are ordered oldest-to-newest and returned as a list under `data.evaluations`; without it, only the single most recent evaluation is returned under `data`.
