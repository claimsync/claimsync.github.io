# CMR Reviews

This module manages the Comprehensive Medication Review (CMR) workflow for a patient, supporting Medicare Part D MTM requirements. It tracks reviewer sign-off on nine Jerimed clinical analyses (feeding the C08 quality gap), and stores the pharmacist's medication reconciliation ("Confirm & Save") record, including active conditions, medication actions, and reconciliation answers.

## Endpoints

| Method | Path | Purpose |
|---|---|---|
| POST | `/webapi/api/v1/patients/{patient_id}/cmr-reviews` | Mark or unmark one of the 9 Jerimed analyses as reviewed; returns full completion state |
| GET | `/webapi/api/v1/patients/{patient_id}/cmr-reviews` | Return CMR analysis review completion state (`all_complete: true` means the C08 gap can close) |
| POST | `/webapi/api/v1/patients/{patient_id}/cmr-reviews/medication-review` | Save CMR medication review selections and reconciliation answers (Confirm & Save); returns 201 |
| GET | `/webapi/api/v1/patients/{patient_id}/cmr-reviews/medication-review` | Return the most recent CMR medication review for a patient; 404 if none exists |

## Data model

| Table | Schema | Stores |
|---|---|---|
| `cmr_analysis_reviews` (`CmrAnalysisReview`) | `quality_measures` | One row per patient with a JSONB column per analysis (`mai`, `acb`, `slm`, `hub`, `clinical_risk`, `opioids`, `fall_risk`, `optimization`, `cascades`), each holding `{reviewed, reviewed_by, reviewed_at}`, plus `updated_at` |
| `cmr_medication_reviews` (`CmrMedicationReview`) | `quality_measures` | One row per Confirm & Save action: JSONB `active_conditions` (name/description), JSONB `medications` (name, dose, freq, route, prn, suggestion, citations, action), JSONB `reconciliation` (yes/no + free-text discrepancy answers), `confirmed_by`, `confirmed_at` |

## Notes

- `cmr_analysis_reviews` is effectively a single upsert-able row per patient with one column per analysis; `upsert_analysis` builds raw SQL dynamically using the column name (`analysis_id`), guarded only by a whitelist check (`ALL_ANALYSES`) before interpolation into the `UPDATE`/`INSERT` statement text.
- Unmarking a review (`reviewed=False`) clears `reviewed_by` and `reviewed_at` back to `None` rather than preserving history — there's no audit trail of prior review/unreview cycles.
- `all_complete` in `GET .../cmr-reviews` is computed by requiring all 9 hardcoded analysis keys to be `reviewed=True`; if the patient has no row yet, all analyses default to unreviewed.
- Medication reviews are append-only (`insert_medication_review` always does a fresh `INSERT`); `GET .../medication-review` and the response after a save both resolve to the single latest row via `ORDER BY confirmed_at DESC LIMIT 1`, so prior submissions remain in the table but aren't otherwise exposed via this API.
- `save_medication_review` (POST) always returns HTTP 201 even though it's logically an upsert-by-append pattern; there is no endpoint to fetch review history, only the latest record.
