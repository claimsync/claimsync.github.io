# Recommendations

This module implements a maker-checker clinical workflow: a "maker" (e.g. a physician) creates a **recommendation** proposing actions on a patient's medications, addressed to a specific "checker" (e.g. a specialist/reviewer). The checker accepts, rejects, or modifies each medication individually, both parties can exchange follow-up comments per medication, and the system tracks read/notification state throughout. Users can also save incomplete recommendations as **drafts** to resume later. The module has no top-level path prefix; each of its 5 sub-routers (recommendations, drafts, notifications, patient medications, recipients) defines its own path directly under `/webapi/api/v1`.

## Endpoints

### Recommendations

| Method | Path | Purpose |
|---|---|---|
| POST | `/webapi/api/v1/recommendations` | Create a new recommendation (maker → checker) with one or more medications; notifies the checker |
| GET | `/webapi/api/v1/recommendations/checker/{checker_id}/unread` | List recommendations assigned to a checker where no medication has any activity yet (fresh inbox) |
| PATCH | `/webapi/api/v1/recommendations/{recommendation_id}/medications/{medication_id}/action` | Checker accepts/rejects/modifies a single medication; updates recommendation status; notifies maker if now complete |
| PATCH | `/webapi/api/v1/recommendations/medications/bulk-action` | Checker acts on multiple medications across recommendations in one call |
| GET | `/webapi/api/v1/recommendations/inbox/{user_id}` | List medications received by a user (as receiver in activities), with filters for unread/status/patient |
| GET | `/webapi/api/v1/recommendations/sent/{user_id}` | List medications sent by a user (as sender in activities), with status filter |
| GET | `/webapi/api/v1/recommendations/awaiting-response/{user_id}` | List sent medications not yet read by the receiver within the last N days (default 15) |
| PATCH | `/webapi/api/v1/recommendations/activities/{activity_id}/read` | Mark a specific activity as read |
| POST | `/webapi/api/v1/recommendations/{recommendation_id}/medications/{medication_id}/comments` | Add a follow-up comment on a medication (maker↔checker); notifies the other party |
| GET | `/webapi/api/v1/recommendations/{recommendation_id}/medications/{medication_id}/conversation` | Get full chronological conversation (creation + all activities/comments) for a medication |

### Drafts

| Method | Path | Purpose |
|---|---|---|
| POST | `/webapi/api/v1/draft-recommendations/` | Create a draft recommendation (only `maker_id` required) |
| GET | `/webapi/api/v1/draft-recommendations/{draft_id}` | Get a single draft by ID |
| GET | `/webapi/api/v1/draft-recommendations/user/{maker_id}` | Paginated list of a maker's draft summaries |
| PATCH | `/webapi/api/v1/draft-recommendations/{draft_id}` | Update a draft (at least one field required) |
| DELETE | `/webapi/api/v1/draft-recommendations/{draft_id}` | Delete a draft |

### Notifications

| Method | Path | Purpose |
|---|---|---|
| GET | `/webapi/api/v1/notifications/users/{firebase_uid}` | List notifications for a user, enriched with sender/recipient names and type-specific fields |
| PATCH | `/webapi/api/v1/notifications/{notification_id}` | Update a notification's `is_read`/`is_dismissed` flags |
| PUT | `/webapi/api/v1/notifications/read` | Bulk-mark all unread notifications as read for a given user + recommendation |

### Patient medications

| Method | Path | Purpose |
|---|---|---|
| GET | `/webapi/api/v1/patient/{patient_id}/medications` | List a patient's active medications (from the latest analysis run) in simple `{id, name, dosage}` form for selection dropdowns |

### Recipients

| Method | Path | Purpose |
|---|---|---|
| GET | `/webapi/api/v1/recipients/` | Paginated list of jerimed users eligible to receive recommendations (excludes current user; filterable by org/role/status) |

## Data model

- **`recommendation.recommendations`** (`Recommendation`) — `id` (UUID), `maker_id`, `checker_id`, `patient_id` (all stored as Firebase UIDs / patient IDs), optional `subject`, `overall_status`, timestamps. Has many `medications` and `notifications`.
- **`recommendation.recommendation_medications`** (`RecommendationMedication`) — one row per medication on a recommendation: `recommendation_id` (FK), `medication_id` (points to `patient_medications` or a custom ID), `is_custom`/`custom_medication_name`, `medication_name`, `clinical_recommendation` (JSONB), `recommended_action`, `specialist_approval_required`, checker-side `action_status`, `checker_comments`, `actioned_at`.
- **`recommendation.recommendation_medication_activities`** (`RecommendationMedicationActivity`) — auto-increment log of every action/comment on a medication: `recommendation_medication_id`, `sender_id`, `receiver_id`, `comments`, `is_read`, `created_at`. Both checker actions and follow-up comments are recorded here.
- **`recommendation.notifications`** (`Notification`) — `id`, `user_id` (recipient), `recommendation_id` (FK), optional `medication_id`, `notification_type`, `is_read`/`read_at`, `is_dismissed`/`dismissed_at`, `created_at`.
- **`recommendation.draft_recommendations`** (`DraftRecommendation`) — `id`, `maker_id` (only required field), optional `checker_id`/`patient_id`/`subject`, `medications` stored as a raw JSON blob (not normalized rows), timestamps.
- **`public.patient_medications`** (`PatientMedication`) — read-only reference table of a patient's medications per analysis `run_id` (dose, route, frequency, PRN flag, active/stop dates, `status`), used to resolve names/dosages shown in recommendations.

**Enum values (`v1/models/enums.py`):**

- `RecommendationStatus`: `pending`, `in_review`, `completed`
- `ActionStatus`: `pending`, `accepted`, `rejected`, `modified`
- `NotificationType`: `new_recommendation`, `recommendation_response`, `recommendation_modified`, `follow_up_comment`

## Notes

- **Maker-checker flow**: creating a recommendation validates that both `maker_id` and `checker_id` exist as Firebase UIDs, requires ≥1 medication, and auto-derives `patient_id` from the first non-custom medication if not supplied (400 error if all medications are custom and no `patient_id` given). A `new_recommendation` notification is created for the checker on success.
- **Status transitions are derived, not set directly**: after any medication action (single or bulk), the recommendation's `overall_status` is recalculated from the pending/total medication counts — `pending` (none actioned), `in_review` (some pending), `completed` (none pending). Completion triggers a `recommendation_response` notification back to the maker.
- **IDs are dual-purpose**: several endpoints accept either the internal `RecommendationMedication` UUID or the underlying `patient_medication_id` and resolve accordingly (`get_medication_by_id` then falls back to `get_medication_by_recommendation_and_patient_medication`). Sender/receiver IDs on activities are normalized to Firebase UIDs via lookup helpers, falling back to the raw stored value if conversion fails.
- **Follow-up comments are authorization-gated**: only the recommendation's maker or checker can post a comment (403 otherwise); the system infers sender/receiver automatically (maker→checker or checker→maker) and creates a `follow_up_comment` notification for the other party.
- **Conversation history is synthesized, not stored as a single object**: it stitches together a synthetic "recommendation_created" item plus every `RecommendationMedicationActivity` row for that medication, and back-fills `action_status` on an activity only if its timestamp is within 5 seconds of the medication's `actioned_at`.
- **Drafts have no maker-checker semantics** — they're a free-form staging area (JSON blob of medications, everything but `maker_id` optional) with no notifications, status, or activities; nothing links a draft to a submitted recommendation once it's created via `POST /recommendations`.
- **Notifications carry denormalized, type-specific display data** computed on read (not stored): human-friendly relative timestamps ("2 hours ago"), `senderName`/`recipientName`, and `hasComments`/`action` fields whose meaning depends on `notification_type`.
