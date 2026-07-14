# Anticholinergic Burden (ACB)

This module retrieves stored ACB (Anticholinergic Cognitive Burden) evaluations for a patient — either the most recent evaluation or the full history — enriching the medication list with PRN status and per-medication risk levels, and caching the single-evaluation response for fast repeat lookups.

## Endpoints

| Method | Path | Purpose |
|---|---|---|
| GET | `/webapi/api/v1/acb/evaluations/{patient_profile_id}` | Retrieve ACB evaluation(s) for a patient. Returns the latest evaluation by default, or all evaluations when `?all=true`. Supports a `validated_filter`-derived query param to restrict results to validated evaluation runs. |

## Data model

`ACBEvaluation` (table: `acb_evaluations`):

- `id` — primary key, auto-incrementing
- `patient_profile_id` — string, indexed, patient profile identifier (e.g. `P-009`)
- `total_acb_score` — numeric, total anticholinergic burden score
- `risk_level` — string, risk classification (`Unknown`, `Low`, `Moderate`, `High`)
- `acb_report` — JSON, detailed report containing `acb_score.medication_details`, `medications`, and `recommendations`
- `created_at` — timestamp (server default `CURRENT_TIMESTAMP`)
- `run_id` — UUID, links the evaluation to a batch processing run (joined against `EvaluationRun`)
- `overridden` / `overridden_at` — boolean + timestamp for manual override tracking

## Notes

- Results are scoped to `EvaluationRun`s with `status == "completed"`; when `validated_only` is true (derived from the `validated_filter` dependency), only runs with `validated == True` are included.
- The single-evaluation path (`all=false`) is cached via `get_cache_key`/`get_from_cache`/`save_to_cache` with a 120-second timeout, keyed by `acb_evaluations:{patient_profile_id}`; cache hits set `from_cache: True` in the response. The `all=true` path is never cached.
- `total_acb_score` and `risk_level` in the response are recomputed from `acb_report.acb_score.total_acb_score` via `get_acb_risk_text` rather than read directly from the `total_acb_score`/`risk_level` columns.
- Medication details in `acb_report` are enriched at read time with `is_prn` (via a lookup against `PatientMedication`, which is optional — if that model can't be imported, PRN enrichment is silently disabled) and with a per-medication `risk_level` derived from `acb_score` (0=None, 1=Low, 2=Moderate, 3=High), only when not already present.
- `cached_helper.py` defines `lru_cache`-memoized helpers (`get_acb_category`, `get_clinical_interpretation`, `get_clinical_significance`) for mapping ACB scores to human-readable category/interpretation/significance text, plus a `serialize_datetime` helper; only `serialize_datetime` is actually used by the service.
- If no evaluations are found, the endpoint raises a 404 with a structured `ErrorResponse` body; any other unexpected error is caught in the router and converted to a 500 with a generic message (the underlying exception is only logged, not exposed to the client).
