# Fall Risk Assessment API

This page documents the API endpoints used to map fall risk data for a patient. All endpoints are part of the `fall-risk-assessment` module under the middleware API.

**Base URL:** `https://middleware-api.claimsync.ai/jerimed-dev/webapi/api/v1/fall-risk-assessment`

---

## 1. Fall History / Overview

Returns the patient's overall fall risk profile, including current and projected medication burden scores, fall history, and IDT (Interdisciplinary Team) decisions.

**Endpoint**

```
GET /overview/{PatientID}
```

**Path Parameters**

| Parameter   | Type   | Description                       |
| ----------- | ------ | --------------------------------- |
| `PatientID` | string | Unique identifier for the patient |

**Response Fields**

| Field                 | Description                                                    |
| --------------------- | --------------------------------------------------------------- |
| `risk_level`          | Overall fall risk classification (e.g., Low / Moderate / High) |
| `current_acb_score`   | Current Anticholinergic Cognitive Burden score                 |
| `projected_acb_score` | Projected ACB score after suggested interventions              |
| `current_slm_score`   | Current Sedative Load Model score                              |
| `projected_slm_score` | Projected SLM score after suggested interventions              |
| `current_cns_score`   | Current CNS depressant burden score                            |
| `projected_cns_score` | Projected CNS score after suggested interventions              |
| `fall_history`        | List of past fall incidents (dates, severity, context)         |
| `executive_summary`   | Narrative summary of the patient's fall risk status            |
| `idt_decisions`       | Decisions/notes recorded by the Interdisciplinary Team         |

---

## 2. Suggested Summary (Medication Plan)

Returns the recommended medication plan changes tied to fall risk, including top contributing medications and a sequenced action plan.

**Endpoint**

```
GET /med-plan/{PatientID}
```

**Path Parameters**

| Parameter   | Type   | Description                       |
| ----------- | ------ | --------------------------------- |
| `PatientID` | string | Unique identifier for the patient |

**Response Fields**

| Field               | Description                                                             |
| ------------------- | ------------------------------------------------------------------------ |
| `executive_summary` | Suggested summary narrative for the medication plan                     |
| `top_contributors`  | Medications/factors contributing most to fall risk                      |
| `sequenced_actions` | Ordered list of recommended actions (e.g., taper, discontinue, monitor) |
| `run_id`            | Identifier for the specific analysis run that generated this plan       |

---

## 3. Care Team

Returns the monitoring schedule and specialist referrals required based on the patient's fall risk assessment.

**Endpoint**

```
GET /care-team/{PatientID}
```

**Path Parameters**

| Parameter   | Type   | Description                       |
| ----------- | ------ | --------------------------------- |
| `PatientID` | string | Unique identifier for the patient |

**Response Fields**

| Field                  | Description                                      |
| ---------------------- | -------------------------------------------------- |
| `monitoring_schedule`  | Recommended frequency/type of monitoring         |
| `specialists_required` | List of specialists recommended for consultation |

---

## 4. Education Context

Returns patient-facing education points and clinical context to support fall risk education and communication.

**Endpoint**

```
GET /education-context/{PatientID}
```

**Path Parameters**

| Parameter   | Type   | Description                       |
| ----------- | ------ | --------------------------------- |
| `PatientID` | string | Unique identifier for the patient |

**Response Fields**

| Field              | Description                                            |
| ------------------ | -------------------------------------------------------- |
| `education_points` | Key education talking points for the patient/caregiver |
| `clinical_context` | Supporting clinical context/rationale                  |

---

## 5. Medication Recommendations

Returns medication optimization recommendations tied to fall risk. This endpoint is used by both the **Implementation** tab (to show specialist requirements and expected score reductions) and the **Care Communication** tab (to show patient-facing explanations for each medication change).

**Endpoint**

```
GET /webapi/api/v1/medication-optimization-recommendations/{PatientID}
```

**Path Parameters**

| Parameter   | Type   | Description                       |
| ----------- | ------ | --------------------------------- |
| `PatientID` | string | Unique identifier for the patient |

**Response Fields**

| Field                       | Description                                                        |
| --------------------------- | -------------------------------------------------------------------- |
| `specialist_required`       | Specialist needed to act on this medication recommendation           |
| `expected_acb_reduction`    | Expected reduction in ACB score if the recommendation is followed     |
| `expected_slm_reduction`    | Expected reduction in SLM score if the recommendation is followed     |
| `why_now`                   | Patient-facing explanation of why this change is being suggested now |
| `what_to_expect`            | Patient-facing explanation of what to expect during the change       |
| `call_us_if`                | Patient-facing guidance on when to contact care team                 |

> **Note:** This same endpoint is called independently by both the Implementation tab and the Care Communication tab.

---

## 6. Clinical Schedule Checklist

Returns the phase-by-phase checklist tasks for a patient's fall risk implementation plan. Used by the **Implementation** tab.

**Endpoint**

```
GET /webapi/api/v1/clinical-schedule/{PatientID}/checklist?run_id={run_id}
```

**Path Parameters**

| Parameter   | Type   | Description                                                  |
| ----------- | ------ | -------------------------------------------------------------- |
| `PatientID` | string | Unique identifier for the patient                              |
| `run_id`    | string | Identifier for the specific analysis run (from the medication plan endpoint) |

**Response Fields**

| Field              | Description                                    |
| ------------------ | ------------------------------------------------ |
| `phaseChecklists`  | Checklist tasks organized by implementation phase |
| `completedTasks`   | Tasks that have already been marked complete      |

> **Related call:** Individual tasks are marked complete via `PATCH /webapi/api/v1/clinical-schedule/checklist/{itemId}`, triggered when a user checks off a task in the Implementation tab.

---
