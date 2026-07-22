# Alerts

Provides read/list access to clinical alerts (CRITICAL, priority=1 medication/optimization recommendations and fall-risk alerts) captured per-tenant by the scheduler, plus endpoints to mark alerts as read, update their lifecycle status, and manually trigger the alert-generation job.

## Endpoints

| Method | Path | Purpose |
|---|---|---|
| GET | `/webapi/api/v1/alerts/` | List alerts for the current tenant, filterable by alert_id, patient_id, run_id, module, status, center_id, with pagination (page/page_size); returns per-alert read status, patient name/age/gender, and total/unread counts |
| POST | `/webapi/api/v1/alerts/refresh` | Trigger the alerts job immediately — for a single tenant (`db_name` query param) or all tenants (no param) |
| PATCH | `/webapi/api/v1/alerts/{alert_id}/status` | Update an alert's status (e.g. to "acknowledged"), recording acknowledged_at/acknowledged_by when acknowledged |
| POST | `/webapi/api/v1/alerts/{alert_id}/read` | Mark an alert as read by the current user (appends firebase_uid to read_by) |

## Data model

`Alert` (table `alerts`, one per tenant DB; unique on `patient_id` + `medication_name` + `module` + `run_id`):

- **Identifiers**: `id` (UUID PK), `patient_id`, `run_id`, `module` (default `"optimization"`)
- **Content**: `medication_name`, `priority`, `action`, `sequencing`, `sequencing_urgency`, `gcn_seqno`, `rationale`, `specialist_approval_required`, `specialist_specialty`
- **Lifecycle**: `status` (default `"new"`), `acknowledged_at`, `acknowledged_by`
- **Read tracking**: `read_by` — array of Firebase UIDs who have read the alert
- **Source/timing**: `source_id`, `alerted_at` (defaults to `CURRENT_TIMESTAMP`)

## Notes

- List/detail queries join each alert to the latest completed run per patient (via a `ROW_NUMBER` subquery over `EvaluationRun`, optionally filtered to `validated = true`) — alerts whose `run_id` isn't the patient's latest matching run are excluded from results.
- Results are ordered with `status == "new"` first, then unread before read, then most recent `alerted_at`; a window `count(*) over()` supplies the `total` for pagination.
- "Read" is per-user: computed as `read_by @> {firebase_uid}` (Postgres array containment), not a global flag.
- `mark_as_read` and `update_status` return 404 if the alert ID doesn't exist.
- `refresh_alerts` bypasses the scheduler's normal cadence and directly imports/calls `_process_optimization_alerts`, `_process_fall_risk_alerts`, and `run_alerts_job` from `webapi.scheduler.scheduler_jobs.optimization_alert`; per-tenant refresh looks up the engine via `get_all_tenant_engines()` and 404s if `db_name` isn't a known/initialized tenant.
