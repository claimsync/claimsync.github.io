# Hub Analysis

Retrieves stored clinical/medication risk analysis results (produced by evaluation runs) for a patient profile, enriching drug profile entries with geriatric risk flags (Beers/HEDIS/STOPP criteria and organ-system risk indicators) sourced from FDB. Exposes both the full list of analyses across all evaluation runs and a condensed summary of the latest run.

## Endpoints

| Method | Path | Purpose |
|---|---|---|
| GET | `/webapi/api/v1/hub/analysis/{patient_profile_id}` | Return all hub analysis records (across all completed evaluation runs) for a patient, each enriched with geriatric flags |
| GET | `/webapi/api/v1/hub/analysis/{patient_profile_id}/summary` | Return a condensed summary (key metrics, categories, recommendations, alerts) of the patient's latest completed evaluation run's hub analysis |

## Data model

**`hub_analysis` table** (`HubAnalysis` model):

| Field | Type | Notes |
|---|---|---|
| `id` | int (PK) | Auto-generated |
| `patient_profile_id` | str | Indexed, required |
| `run_id` | UUID | FK to `evaluation_runs.id`, indexed, nullable |
| `analysis_results` | JSON (dict) | Nullable; holds the raw analysis payload (score, risk level, medications, categories, recommendations, alerts, drug_profiles, etc.) |
| `created_at` | datetime | Defaults to UTC now |
| `overridden` | bool | Defaults to `False`, nullable |
| `overridden_at` | datetime (timezone-aware) | Nullable |

Related read-only dependency: `EvaluationRun` (from `webapi.shared.models.evaluation_run`), filtered by `status == "completed"` and, for non-reviewer roles, `validated == True`.

## Notes

- **Caching**: Both endpoints cache their full response (5-minute TTL, `CACHE_TIMEOUT = 300`) under keys namespaced by `hub_analysis_all_`/`hub_analysis_summary_` + DB/auth identifiers + `patient_profile_id`. Cache hits are flagged by adding `"from_cache": True` to the returned dict.
- **Role-based visibility**: Users whose role grants the `validate_evaluation_run` permission (`ROLE_PERMISSIONS`) are treated as reviewers and can see evaluation runs regardless of `validated` status; non-reviewers only see `validated == True` runs.
- **Geriatric flag enrichment**: The service recursively walks `analysis_results` for any nested `drug_profiles` lists, batches all `gcn_seqno` values into a single query against an external FDB (First Databank) Postgres source (`AlternativeDataFetcher`), and attaches a `geriatric_flag` dict (Beers/HEDIS/STOPP flags, severity level, and renal/hepatic/cardiac/neuro/pulmonary/endocrine risk booleans) to each drug profile entry. If the FDB connection/import fails, entries fall back to all-false/`None` default flags, and the rest of the payload is returned unaffected (enrichment failures are logged, not raised).
- **Error handling**: DB connectivity is explicitly tested (`test_db_connection`) before querying, raising `503` on `OperationalError`/`DBAPIError`. Missing evaluation runs or missing hub analysis records both raise `404` with a structured `{success, error, message}` body. An invalid/empty `patient_profile_id` raises `400`. Router-level catch-all wraps any unexpected exception as a `500` with a generic message.
- The `/summary` endpoint only uses the single latest completed run (`get_latest_evaluation_run`, ordered by `created_at desc`), while the base endpoint aggregates across *all* completed runs for the patient.
