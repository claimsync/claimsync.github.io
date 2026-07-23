# Optimization — API Reference

This page documents the API calls that power the Optimization feature (`src/components/optimization/`), and which of the three tabs — **Triage**, **Implementation**, **Care Communication** — each one belongs to.

---

## 1. Medication Recommendations

Returns all medication recommendations for a patient. Fired once when the page loads, before any tab is opened. The result (`allMedications`) is passed down to all 3 tabs.

**Endpoint**

```
GET /webapi/api/v1/medication-optimization-recommendations/{PatientID}
```

**Path Parameters**

| Parameter   | Type   | Description                       |
| ----------- | ------ | --------------------------------- |
| `PatientID` | string | Unique identifier for the patient |

**Response Fields**

| Field       | Description                                              |
| ----------- | ---------------------------------------------------------- |
| `data`      | List of medications with recommendations                  |
| `run_id`    | Identifier for this analysis run — needed by other calls below |

---

## 2. Recommendation Status 

Returns counts of medications by status. Powers the **Pending**, **Approved**, **Declined**, and **Needs discussion** stat tiles in the header.

**Endpoint**

```
GET /webapi/api/v1/medications/{patient_id}/recommendation-kpi?run_id={run_id}
```

**Path & Query Parameters**

| Parameter    | Type   | Description                                         |
| ------------ | ------ | ------------------------------------------------------ |
| `patient_id` | string | Unique identifier for the patient                    |
| `run_id`     | string | Analysis run ID (from the medication recommendations call above) |

**Response Fields**

| Field                       | Description                          |
| --------------------------- | --------------------------------------- |
| `total_pending`             | Count of medications pending review  |
| `total_approved`            | Count of medications approved        |
| `total_declined`            | Count of medications declined        |
| `total_needs_discussion`    | Count of medications needing discussion |

---

## 3. Checklist Status

Returns overall task-completion counts. Powers the **Ongoing**, **Completed**, and **Tasks completed** stat tiles in the header.

**Endpoint**

```
GET /webapi/api/v1/clinical-schedule/{patient_id}/checklist/kpi?run_id={run_id}
```

**Path & Query Parameters**

| Parameter    | Type   | Description                                         |
| ------------ | ------ | ------------------------------------------------------ |
| `patient_id` | string | Unique identifier for the patient                    |
| `run_id`     | string | Analysis run ID (from the medication recommendations call above) |

**Response Fields**

| Field             | Description                          |
| ----------------- | --------------------------------------- |
| `completed_items` | Number of checklist tasks completed  |
| `total_items`     | Total number of checklist tasks      |

---

## 4. Triage Data

Returns the patient summary, optimization strategy, and burden reduction projections shown on the **Triage** tab. No extra calls are made beyond this — everything else on the tab is derived from data already loaded.

**Endpoint**

```
GET /webapi/api/v1/medication-optimization-triage/{PatientID}
```

**Path Parameters**

| Parameter   | Type   | Description                       |
| ----------- | ------ | --------------------------------- |
| `PatientID` | string | Unique identifier for the patient |

**Response Fields**

| Field                          | Description                                             |
| ------------------------------ | ----------------------------------------------------------- |
| `summary`                      | Patient summary narrative                                |
| `optimization_strategy`        | Overall optimization strategy narrative                  |
| `burden_reduction_projection`  | Current vs. projected medication burden scores           |
| `strategic_decision`           | List of strategic decisions with rationale and medications considered |
| `run_id`                       | Identifier for this analysis run                          |

---

## 5. Clinical Schedule Checklist

Returns the phase-by-phase checklist tasks shown on the **Implementation** tab.

**Endpoint**

```
GET /webapi/api/v1/clinical-schedule/{PatientID}/checklist?run_id={run_id}
```

**Path & Query Parameters**

| Parameter   | Type   | Description                                         |
| ----------- | ------ | ------------------------------------------------------ |
| `PatientID` | string | Unique identifier for the patient                    |
| `run_id`    | string | Analysis run ID (from the medication recommendations call) |

**Response Fields**

| Field                | Description                                    |
| -------------------- | -------------------------------------------------- |
| `data`                | List of checklist items, each with a `primary_key` / `secondary_key` grouping and task details |
| `is_check`            | Whether the task has been marked complete           |
| `checked_at`          | Timestamp the task was checked, if completed        |
| `checked_by_name`     | Name of the user who checked the task, if completed |

---

## 6. Update Checklist Task

Marks a single checklist item as complete or incomplete. Called on the **Implementation** tab when a user checks off a task. After this call succeeds, the checklist (Section 5) is re-fetched to refresh the UI.

**Endpoint**

```
PATCH /webapi/api/v1/clinical-schedule/checklist/{itemId}
```

**Path Parameters**

| Parameter | Type   | Description                       |
| --------- | ------ | ---------------------------------- |
| `itemId`  | string | Unique identifier for the checklist item |

**Request Body**

| Field         | Type    | Description                              |
| ------------- | ------- | ------------------------------------------ |
| `checked_by`  | string  | ID of the user marking the task             |
| `is_check`    | boolean | `true` to mark complete, `false` to undo    |

---

## Care Communication Tab

No additional API calls. This tab renders entirely from the `allMedications` data already loaded in **Section 1** (Shared — Page Load).

---

## Summary Table

| #   | Tab                  | Purpose                        | Method | Endpoint                                                                 |
| --- | --------------------- | -------------------------------- | ------ | ------------------------------------------------------------------------- |
| 1   | Shared (page load)     | Medication Recommendations       | GET    | `/webapi/api/v1/medication-optimization-recommendations/{PatientID}`     |
| 2   | Shared (page load)     | Recommendation Status            | GET    | `/webapi/api/v1/medications/{patient_id}/recommendation-kpi?run_id={run_id}` |
| 3   | Shared (page load)     | Checklist Status                 | GET    | `/webapi/api/v1/clinical-schedule/{patient_id}/checklist/kpi?run_id={run_id}` |
| 4   | Triage                 | Triage Data                      | GET    | `/webapi/api/v1/medication-optimization-triage/{PatientID}`               |
| 5   | Implementation         | Clinical Schedule Checklist      | GET    | `/webapi/api/v1/clinical-schedule/{PatientID}/checklist?run_id={run_id}` |
| 6   | Implementation         | Update Checklist Task            | PATCH  | `/webapi/api/v1/clinical-schedule/checklist/{itemId}`                     |
| —   | Care Communication     | (No additional calls)            | —      | Reuses data from #1                                                        |
