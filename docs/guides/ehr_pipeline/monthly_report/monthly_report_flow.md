# POST /generate-monthly-report — End-to-End Flow

## Overview

This endpoint generates a monthly medication report for a tenant. It runs a 3-part pipeline that resolves patient run IDs, queries evaluation scores, and performs medication reconciliation. It is designed to be triggered by GCP Cloud Scheduler or called manually.

**File:** `services/monthly_report_service.py`

---

## Step 1 — HTTP Request

**Endpoint:** `POST /generate-monthly-report`

**Request body (`MonthlyReportRequest`):**

```json
{
  "tenant": "Sandbox",
  "month": "03/2026",
  "dry_run": true,
  "use_llm": false,
  "limit": null,
  "patient_ids": null
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `tenant` | string | Yes | Tenant name (e.g. `"Sandbox"`, `"PaceNorth"`) |
| `month` | `MM/YYYY` | No | Target month. Defaults to current month. Mutually exclusive with `start_date`/`end_date` |
| `start_date` | `YYYY-MM-DD` | No | Custom range start. Must be used with `end_date` |
| `end_date` | `YYYY-MM-DD` | No | Custom range end (inclusive). Must be used with `start_date` |
| `dry_run` | bool | No | Default `true`. When `true`, no data is written to DB |
| `use_llm` | bool | No | Default `false`. Enables LLM semantic matching in reconciliation |
| `limit` | int | No | Cap patients processed (e.g. `5` for testing) |
| `patient_ids` | list | No | Filter to specific patient IDs |

**Validation rules:**
- `month` and `start_date`/`end_date` cannot both be provided
- If `start_date` is provided, `end_date` is also required (and vice versa)

---

## Step 2 — Tenant normalization + DB connections

**Function:** `generate_report_endpoint()` → `generate_monthly_report()`

1. `normalize_site_name(request.tenant)` normalizes the tenant string
2. Connects to **tenant PostgreSQL DB** using `DatabaseConfigManager.get_pg_config(tenant)`
3. Connects to **FDB** (formulary database) using `DatabaseConfigManager.get_fdb_config()`

---

## Step 3 — Compute date bounds

**Function:** `compute_month_bounds(month, start_date, end_date)`

Converts the input into an ISO datetime range used in SQL queries.

| Input | `month_start` | `month_end` |
|---|---|---|
| `month="03/2026"` | `2026-03-01T00:00:00` | `2026-04-01T00:00:00` |
| `start_date="2026-03-01"`, `end_date="2026-03-31"` | `2026-03-01T00:00:00` | `2026-04-01T00:00:00` |
| No params | First day of current month | First day of next month |

SQL uses: `completed_at >= month_start AND completed_at < month_end`

**DB label computation:**

```python
if start_date and end_date:
    month_label = f"{start_date} to {end_date}"        # used only for logging
    db_month_label = month_start.strftime("%m/%Y")     # stored in DB: "03/2026"
else:
    month_label = month or current_month               # e.g. "03/2026"
    db_month_label = month_label                       # same value stored in DB
```

`db_month_label` (format `MM/YYYY`) is what gets stored in the `month` column of `monthly_report_scores`.

---

## Step 4 — Part 1: Resolve patient run IDs

**Function:** `resolve_monthly_runs(pg_manager, month_start, month_end, limit, patient_ids)`

### 4a. Get all patients

```sql
SELECT DISTINCT patient_profile_id
FROM evaluation_runs
WHERE status = 'completed' AND validated = true
```

If `patient_ids` is provided, filters to only those IDs. If `limit` is set, truncates the list.

### 4b. For each patient, find two runs

**Current run** — latest completed run within the target month:
```sql
SELECT id, status, completed_at
FROM evaluation_runs
WHERE patient_profile_id = %(patient_id)s
  AND status = 'completed' AND validated = true
  AND completed_at >= %(month_start)s::timestamptz
  AND completed_at < %(month_end)s::timestamptz
ORDER BY completed_at DESC
LIMIT 1
```

**Previous run** — latest completed run before the target month:
```sql
SELECT id, status, completed_at
FROM evaluation_runs
WHERE patient_profile_id = %(patient_id)s
  AND status = 'completed' AND validated = true
  AND completed_at < %(month_start)s::timestamptz
ORDER BY completed_at DESC
LIMIT 1
```

### 4c. Output per patient

```python
{
  "patient_id": "P1",
  "current_run_id": "uuid-current",       # None if no run this month
  "current_completed_at": "2026-03-15T...",
  "previous_run_id": "uuid-previous",     # None if first-ever run
  "previous_completed_at": "2026-02-10T..."
}
```

---

## Step 5 — Part 3: Medication reconciliation

**Function:** `run_reconciliation_for_patients(pg_manager, fdb_manager, run_resolutions, use_llm, dry_run)`

Runs before Part 2 so that med counts are available for score summaries.

### 5a. Per patient

For each patient with at least one run:

1. Fetch **current meds** from `patient_medications` (for `current_run_id`, status `ACTIVE`)
2. Fetch **previous meds** from `patient_medications` (for `previous_run_id`, status `ACTIVE`)
3. Convert both to DataFrames

### 5b. Baseline check

If `previous_run_id` is `None` (first-ever run), all current meds are marked `UNCHANGED` as a baseline — no reconciliation is performed.

### 5c. Two-pass reconciliation

**Pass 1 — Exact GCN match:**
- Pairs current and previous meds with identical `gcn_seqno`
- If parameters (dose, frequency, route) are the same → `UNCHANGED`
- If parameters differ → `MODIFIED`

**Pass 2 — Remaining meds via `MedicationReconciler`:**
- Unmatched meds go through HICL-based matching
- If `use_llm=True`, falls back to LLM semantic matching for ambiguous cases
- Unmatched current meds → `NEW`
- Unmatched previous meds → `DISCONTINUED`

### 5d. Per-medication enrichment

For each changed/new/discontinued med, loads:
- ACB score (from `acb_evaluations.acb_report` JSON)
- SLM score (from `slm_evaluations.slm_report` JSON)
- MAI focus criteria (from `mai_evaluations` where rating = `"C"`)

Name lookup tries: full name → normalized base name → GCN → HICL fallback.

### 5e. DB write (if `dry_run=False`)

**Table: `medication_reconciliation_results`**

Behavior on re-run: **always inserts a new row** (no `ON CONFLICT`). Before inserting, the existing active row for `(patient_id, current_run_id)` is marked `overridden=TRUE`:

```sql
-- Step 1: mark old row overridden
UPDATE medication_reconciliation_results
SET overridden = TRUE, overridden_at = NOW()
WHERE patient_profile_id = %(patient_id)s
  AND current_run_id = %(current_run_id)s
  AND overridden = FALSE

-- Step 2: insert new row and return its ID
INSERT INTO medication_reconciliation_results (
  patient_profile_id, current_run_id, previous_run_id,
  new_count, modified_count, discontinued_count, unchanged_count,
  total_medications_current, total_medications_previous,
  reconciliation_report, processing_time_ms, is_baseline
) VALUES (...) RETURNING id
```

The returned `id` (`recon_row_id`) is passed to Part 2 for linking.

### 5f. Output per patient

```python
{
  "patient_id": "P1",
  "current_run_id": "uuid-current",
  "new_count": 2,
  "modified_count": 1,
  "discontinued_count": 0,
  "unchanged_count": 8,
  "total_current": 11,
  "total_previous": 9,
  "is_baseline": False,
  "recon_row_id": 102,           # None if dry_run=True
  "reconciliation_report": {...} # full per-med detail
}
```

---

## Step 6 — Part 2: Query evaluation scores

**Function:** `query_patient_scores(pg_manager, run_resolutions, recon_summaries, db_month_label, dry_run, month_start, month_end)`

### 6a. Pre-checks

- Checks if `patient_falls` table exists
- Checks if `patient_hospitalizations` table exists
- Builds a lookup of `med_counts` and `recon_ids` from Part 3 output

### 6b. Per patient — run-based scores

For each of `current_run_id` and `previous_run_id`, queries:

| Score | Source table | Column |
|---|---|---|
| ACB total | `acb_evaluations` | `total_acb_score`, `risk_level` |
| SLM total | `slm_evaluations` | `total_sedative_load` |
| Fall risk score | `fall_risk_idt_evaluations` | `fall_risk_score`, `fall_risk_level` |
| MAI total, urgency, risk levels, overall risk | `patient_statistics` | `mai_metrics`, `slm_metrics`, `fall_risk_metrics`, `overall_risk_level` |

### 6c. Per patient — month-based events

Falls and hospitalizations are counted from event tables using the month range (not run-based):

| Field | Table | Logic |
|---|---|---|
| `current_falls` | `patient_falls` | `fall_date` within `[cutoff, month_end)` |
| `previous_falls` | `patient_falls` | `fall_date` within `[cutoff, month_start)` |
| `current_hosp` | `patient_hospitalizations` | `admission_date` within `[cutoff, month_end)` |
| `previous_hosp` | `patient_hospitalizations` | `admission_date` within `[cutoff, month_start)` |

`EVENT_CUTOFF_DATE = "2025-06-01"` — events before this date are excluded.

### 6d. DB write (if `dry_run=False`)

**Table: `monthly_report_scores`**

Unique key: `(patient_profile_id, month)`

**On first run:** INSERT new row.
**On re-run:** `ON CONFLICT (patient_profile_id, month) DO UPDATE SET` — all columns are overwritten with the latest values. Always results in exactly 1 row per patient per month.

```sql
INSERT INTO monthly_report_scores (
  patient_profile_id, month, current_run_id, current_run_date,
  previous_run_id, previous_run_date,
  current_med_count, previous_med_count,
  current_acb, previous_acb,
  current_slm, previous_slm,
  current_mai, previous_mai,
  current_falls, previous_falls,
  current_hosp, previous_hosp,
  current_fall_risk_score, previous_fall_risk_score,
  current_fall_risk_level, previous_fall_risk_level,
  current_mai_urgency, previous_mai_urgency,
  current_mai_risk_level, previous_mai_risk_level,
  current_acb_risk_level, previous_acb_risk_level,
  current_slm_risk_level, previous_slm_risk_level,
  current_overall_risk_level, previous_overall_risk_level,
  medication_reconciliation_results_id
) VALUES (...) ON CONFLICT (patient_profile_id, month) DO UPDATE SET ...
```

### 6e. Output per patient

```python
{
  "patient_id": "P1",
  "month": "03/2026",
  "current_run_id": "uuid-current",
  "current_run_date": "2026-03-15T...",
  "previous_run_id": "uuid-previous",
  "previous_run_date": "2026-02-10T...",
  "current_med_count": 11,
  "previous_med_count": 9,
  "current_acb": 4.0,
  "previous_acb": 3.0,
  "current_slm": 2.5,
  "previous_slm": 2.0,
  "current_mai": 6.0,
  "previous_mai": 5.0,
  "current_falls": 1,
  "previous_falls": 0,
  "current_hosp": 0,
  "previous_hosp": 0,
  "current_fall_risk_score": 0.72,
  "current_overall_risk_level": "High",
  ...
}
```

---

## Step 7 — HTTP Response

**Model:** `MonthlyReportResponse`

```json
{
  "status": "success",
  "tenant": "Sandbox",
  "month": "03/2026",
  "generated_at": "2026-03-01T10:00:00",
  "total_patients": 42,
  "part1_run_resolution": [...],
  "part2_score_summaries": [...],
  "part3_reconciliation_summaries": [...]
}
```

---

## Re-run behavior (same month, `dry_run=False`)

| Table | Behavior |
|---|---|
| `monthly_report_scores` | **UPSERT** — 1 row per patient per month, always overwritten |
| `medication_reconciliation_results` | **INSERT new row** — old row marked `overridden=TRUE`, new `recon_id` linked in scores |

---

## Full call chain

```
POST /generate-monthly-report
  └── generate_report_endpoint()                        # validates request
        └── generate_monthly_report()                   # main orchestrator
              ├── connect tenant DB + FDB
              ├── compute_month_bounds()                 # Step 3
              ├── resolve_monthly_runs()                 # Part 1 — Step 4
              │     ├── ALL_PATIENTS_QUERY
              │     ├── MONTHLY_CURRENT_RUN_QUERY (per patient)
              │     └── MONTHLY_PREVIOUS_RUN_QUERY (per patient)
              ├── run_reconciliation_for_patients()      # Part 3 — Step 5
              │     ├── MEDS_QUERY x2 (current + previous)
              │     ├── _two_pass_reconcile()
              │     │     ├── Pass 1: exact GCN match
              │     │     └── Pass 2: MedicationReconciler (HICL / LLM)
              │     ├── _load_acb_scores() / _load_slm_scores() / _load_mai_scores()
              │     └── _store_recon_result()            # if dry_run=False
              │           ├── OVERRIDE_RECON_RESULT
              │           └── INSERT_RECON_RESULT RETURNING id
              └── query_patient_scores()                 # Part 2 — Step 6
                    ├── _query_run_scores() x2 (current + previous run)
                    │     ├── ACB_SCORES_QUERY
                    │     ├── SLM_SCORES_QUERY
                    │     ├── FALL_RISK_QUERY
                    │     └── PATIENT_STATS_QUERY
                    ├── _query_patient_falls_hosp()
                    │     ├── FALLS_CURRENT_QUERY + FALLS_PREVIOUS_QUERY
                    │     └── HOSP_CURRENT_QUERY + HOSP_PREVIOUS_QUERY
                    └── _store_monthly_score()           # if dry_run=False
                          └── INSERT_MONTHLY_SCORE (ON CONFLICT DO UPDATE)
```

---

## Key tables written

| Table | Written in | Key | On re-run |
|---|---|---|---|
| `medication_reconciliation_results` | Part 3 | `(patient_profile_id, current_run_id)` | New row inserted, old row marked `overridden=TRUE` |
| `monthly_report_scores` | Part 2 | `(patient_profile_id, month)` | Existing row updated (UPSERT) |
