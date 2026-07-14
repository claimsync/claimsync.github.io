# Cascade

Retrieves prescribing-cascade hypotheses for a patient — cases where a chain of medications suggests one drug was prescribed to treat a side effect of another — from the patient's latest completed evaluation run.

## Endpoints

| Method | Path | Purpose |
|---|---|---|
| GET | `/webapi/api/v1/cascade/evaluations/{patient_id}` | Get the cascade hypotheses JSON for a patient's latest completed evaluation run |

## Data model

- `cascade_evaluations` table (queried via raw SQL, not a full ORM model): `patient_profile_id`, `run_id`, `overridden` (boolean flag; only rows where `overridden = FALSE` are returned), `cascade_hypotheses` (JSON/JSONB column, may be returned as a string and is parsed with `json.loads`).
- Each item in `cascade_hypotheses` is expected to have: `chain` (list of drug entries, each with a `gcn` code), `pattern_name` (slug used to look up a known cascade pattern), and optionally `clinical_consequence`.
- Response schema (`cascade_schema.py`):
  - `CascadeHypothesesResponse`: `success: bool`, `data: CascadeData`
  - `CascadeData`: `patient_id: str`, `run_id: str`, `cascade_hypotheses: Any`

## Notes

- Per-tenant kill switch: if the request's tenant (`request.state.tenant_db_name`) is in `CASCADES_SUPPRESSED_TENANTS` (from `webapi.core.config`), the endpoint short-circuits and returns an empty result (`run_id: None`, `cascade_hypotheses: []`) instead of querying the database.
- The service first resolves the patient's latest completed run via `get_latest_evaluation_run_id` (shared clinical-utils helper) using a `validated_filter`; if no run is found, or if the repo finds no matching row, the router returns a 404.
- Hypotheses are deduplicated by the frozenset of `gcn` codes in each hypothesis's `chain` — duplicate drug-chain combinations are collapsed, keeping only the first occurrence.
- Hypotheses are enriched using `SLUG_TO_PATTERN` (from `core_business_logic.src.workflows.cascade_patterns`): if a hypothesis lacks `clinical_consequence`, it's backfilled from the matched pattern's `name`, and a `constant_sources` field is always added from the pattern's `sources`.
- Error handling is uniform: any unexpected exception is logged with a stack trace and converted to a 500 with a generic `{"success": false, ...}` body; `HTTPException`s (e.g., the 404) are re-raised as-is.
