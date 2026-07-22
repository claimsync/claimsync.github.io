# Hospitalization (PACE Discharge Review)

This module exposes a read-only API for retrieving PACE program hospital discharge review records for a patient, backed by JSON-based discharge data stored per patient/run.

## Endpoints

| Method | Path | Purpose |
|---|---|---|
| GET | `/webapi/api/v1/pace-discharge-review/{patient_id}` | Get all discharge reviews for a given patient, ordered by most recent (highest id) first |

## Data model

**`pace_discharge_review`** (`PaceDischargeReview`)

| Field | Type | Notes |
|---|---|---|
| `id` | int | Primary key, auto-generated |
| `patient_id` | str | Indexed; identifies the patient the review belongs to |
| `run_id` | str, optional | Identifier for the processing run that produced the review |
| `hospital_discharge_data_json` | JSON, optional | Freeform structured discharge data payload |

## Notes

- The top-level `router.py` mounts with an empty prefix, so the effective path is defined entirely by `v1/router.py`, which combined with the app's `/webapi/api/v1` base gives the full path `/webapi/api/v1/pace-discharge-review/{patient_id}`.
- The service layer returns a 404 (with `success: false`, `data: []`, `count: 0`) when no discharge reviews exist for the patient, rather than returning an empty 200 response.
- On success, the response is wrapped in an envelope: `{"success": true, "data": [...], "count": N, "message": "..."}`.
- Both the repository and service raise `HTTPException` on DB errors (500), and the router explicitly re-raises `HTTPException` as-is while wrapping any other unexpected exception in a generic 500 response — so error detail structure is consistent (`success`, `error`, `message` keys) across layers.
- Results are ordered by `id` descending, so the most recently created review row is returned first; there is no filtering by `run_id` or pagination support.
