# Medication Appropriateness Index (MAI)

This module exposes patient MAI evaluation history — a set of AI-generated, weighted appropriateness ratings (Indication, Effectiveness, Dosage, etc.) computed per medication and grouped into evaluation "runs". Only one of the two implementations (v1 or v2) is mounted at a time, selected by the `IS_NEW_MAI` environment variable in `router.py`, and both expose the exact same route path once active.

## Endpoints

### v1

| Method | Path | Purpose |
|---|---|---|
| GET | `/webapi/api/v1/mai/evaluations/{patient_profile_id}/all_mai` | Get all MAI evaluation runs for a patient, paginated, from the `mai_wiki_evaluation` table (`include_medications`, `limit` max 15 default 15, `offset` query params) |

### v2

| Method | Path | Purpose |
|---|---|---|
| GET | `/webapi/api/v1/mai/evaluations/{patient_profile_id}/all_mai` | Get all MAI evaluation runs for a patient, paginated, from the `mai_evaluations` table (`include_medications`, `limit` max 15 default 15, `offset` query params) |

## Data model

| Model | Version | Table | Key fields |
|---|---|---|---|
| `MaiWikiEvaluation` | v1 | `mai_wiki_evaluation` | `id`, `patient_profile_id`, `run_id` (UUID), `gcn_seqno`, `medication_name`, `q1_result`…`q9_result` (JSONB per-criterion assessment), `created_at`, `overridden`, `overridden_at` |
| `MaiEvaluation` | v2 | `mai_evaluations` | `id`, `patient_profile_id`, `run_id` (UUID), `gcn_seqno`, `medication_name`, `q1_result`…`q9_result` (JSONB per-criterion assessment), `created_at`, `overridden`, `overridden_at` |

Both models are structurally identical (same columns/types) — only the table name and field docstrings differ.

## Notes

- Versioning is not URL-based: `webapi/mai/router.py` mounts either the v1 or v2 sub-router under the same `/evaluations/{patient_profile_id}/all_mai` path based on `IS_NEW_MAI` — when `IS_NEW_MAI=true` the **v1** router is active, otherwise (default) **v2** is active. The two versions are never live simultaneously.
- Per [ClaimSync's stated v2 convention](../CLAUDE.md#how-to-add-a-v2-to-an-existing-module), v2 should reuse v1 models rather than duplicating them — it does not. `v2/models/mai_evaluation.py` and `v2/schemas/response.py` are near line-for-line duplicates of the v1 versions (same field sets, just renamed classes), and the scoring logic (`CRITERION_MAPPING`, `CRITERION_WEIGHTS`, `_calculate_mai_score`, `_build_q_entry`, `_extract_rating`) in `v1/services/mai_service.py` and `v2/services/mai_service.py` is duplicated verbatim rather than shared.
- Run identity differs subtly between versions: v1's service groups/labels runs using the dedicated `row.run_id` UUID field; v2's service instead uses `row.id` (the primary key) as the value reported as `run_id`.
- v1 has a fallback for `mean_mai_score`: if `get_mean_mai_score_sqlmodel` returns `None`, it computes an average from the medications' `mai_score.total_score` values. v2 has no such fallback — it will return `mean_mai_score`/`mean_risk_text` as `None` in that case.
- Pagination is identical in both versions: `limit` (default 15, max 15 via `le=15`), `offset` (default 0), with computed `total_pages`, `current_page`, `has_next`/`has_previous`, `next_offset`/`previous_offset` in the response. `validated_only` filtering is driven by the `get_validated_filter` dependency (non-empty filter value means validated-only).
- Error handling is duplicated identically in both routers: any exception whose message contains `"UndefinedTable"` or `"does not exist"` is treated as a missing per-tenant table and returns 404 with a structured `{success, error, message}` body; all other exceptions are logged and returned as a generic 500 with the same structured body shape.
