# Sedative Load Model (SLM)

This module retrieves stored SLM evaluations — a per-patient sedative burden score computed from a patient's medications — supporting a single "latest evaluation" view or a full history, with medication-level risk detail, PRN enrichment, and short-lived response caching.

## Endpoints

| Method | Path | Purpose |
|---|---|---|
| GET | `/webapi/api/v1/slm/evaluations/{patient_profile_id}` | Retrieve SLM evaluation(s) for a patient. Returns the latest evaluation by default, or all evaluations when `?all=true`. Validation scope is controlled via the `get_validated_filter` dependency. |

## Data model

`SLMEvaluation` (table `slm_evaluations`):

- `id` — primary key (autoincrement)
- `patient_profile_id` — string(100), indexed, not null
- `total_sedative_load` — Numeric(10,2), not null (the raw SLM score)
- `medications_count` — integer, not null
- `slm_report` — JSONB, nullable (full report payload, including `slm_score.medication_details`/`medication_details` and `recommendations`)
- `created_at` — timezone-aware datetime, defaults to `utcnow`
- `run_id` — string(36), indexed, nullable (links to `EvaluationRun`)
- `overridden` — boolean, nullable, default `False`
- `overridden_at` — timezone-aware datetime, nullable

Indexes: `ix_patient_created` (patient_profile_id, created_at), `ix_sedative_load` (total_sedative_load).

Computed property `risk_level`: Low (<5), Moderate (<10), High (>=10) based on `total_sedative_load` — note this differs from the `get_slm_risk_text` thresholds used elsewhere in the service layer.

## Notes

- Only evaluations whose `run_id` belongs to an `EvaluationRun` with `status == "completed"` are ever returned; when `validated_only` is true (derived from the `get_validated_filter` dependency), the run must also have `validated == True`.
- For a single (non-`all`) request, the service checks a cache (`slm_evaluations:{patient_profile_id}_{validated_only}`) before hitting the DB, and writes the built response back to cache with a 120-second timeout; cache hits are flagged with `from_cache: True` in the response.
- A `sedative_rating` of exactly `0.5` on any medication is silently normalized to `0` before returning, and the deducted amount is subtracted from `total_sedative_load`, `slm_score.total_slm_score`, and `recommendations.total_sedative_load` (clamped to a minimum of 0) — this reconciles a legacy half-point rating without mutating the stored ORM data.
- Medication details are enriched with `is_prn` status looked up from `PatientMedication` by `(run_id, gcn_seqno)`; if the `PatientMedication` model can't be imported, PRN enrichment is disabled entirely (logged as a warning) and `is_prn` defaults to `False`.
- Each medication dict also gets a flattened top-level `risk_level` copied from its nested `interpretation.risk_level`, without mutating the original `slm_report` medication entries in place (copies are built instead).
- Unhandled exceptions in the endpoint are caught and converted to a generic 500 `HTTPException` with a standard `{success, error, message}` body; a "no evaluations found" condition raises a 404 with the same shape.
