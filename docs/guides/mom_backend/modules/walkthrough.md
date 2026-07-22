# User Walkthrough

Tracks per-user completion state for product-tour/walkthrough steps across various application features, auto-creating a default (all-incomplete) record on first access and allowing partial updates as a user progresses through the tour.

## Endpoints

| Method | Path | Purpose |
|---|---|---|
| GET | `/webapi/api/v1/jerimed-users/{firebase_uid}/walkthrough` | Fetch a user's walkthrough completion state, creating a default record if none exists |
| PATCH | `/webapi/api/v1/jerimed-users/{firebase_uid}/walkthrough` | Update one or more walkthrough completion flags for a user |

## Data model

`UserWalkthrough` (table: `user_walkthrough`)

- `id` — primary key
- `firebase_uid` — unique, non-null, max length 100; links the record to a user
- Boolean completion flags (all default `false`, non-null): `slm`, `hub`, `acb`, `reconciliation`, `optimization`, `recommendations`, `fall_risk`, `hospitalizations`, `opioid`, `fall_risk_idt`, `mai`, `patient_list`, `cascade`
- `created_at` — set on creation
- `updated_at` — set on creation and refreshed on update

## Notes

- Both endpoints validate that the `firebase_uid` corresponds to an existing user via `JerimedUserRepository` (from the `jerimed_users` module); a missing user raises a `ValueError`, which the router converts to HTTP 404.
- On both GET and PATCH, if no walkthrough record exists yet for the user, one is created on the fly with all flags defaulting to `false` before being read or updated.
- `PATCH` uses `UserWalkthroughUpdate.model_dump(exclude_unset=True)` so only explicitly provided fields are updated; omitted fields are left unchanged.
- The `v1/router.py` sub-router itself has no additional path prefix — the distinguishing path segment is just `/{firebase_uid}/walkthrough` appended directly under `/jerimed-users`. This module shares its top-level prefix with the [Jerimed Users](jerimed-users.md) module but is a separate module/table.
- `updated_at` is refreshed manually in the repository's `update` method (in addition to the SQLModel `onupdate` column kwarg).
