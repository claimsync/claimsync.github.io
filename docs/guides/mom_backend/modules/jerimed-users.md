# Jerimed Users

Manages Jerimed platform user accounts (linked to Firebase authentication) within a tenant database, supporting creation, lookup, listing with pagination/filtering, and full/partial updates.

## Endpoints

| Method | Path | Purpose |
|---|---|---|
| POST | `/webapi/api/v1/jerimed-users/` | Create a new jerimed user |
| GET | `/webapi/api/v1/jerimed-users/` | List jerimed users (paginated, filterable by org_id, role_id, status) |
| GET | `/webapi/api/v1/jerimed-users/{user_id}` | Get a jerimed user by ID |
| PUT | `/webapi/api/v1/jerimed-users/{user_id}` | Full update of a jerimed user |
| PATCH | `/webapi/api/v1/jerimed-users/{user_id}` | Partial update of a jerimed user |

## Data model

**`jerimed_user` table** (`JerimedUser`):

- `id` (int, primary key)
- `firebase_uid` (str, required, unique)
- `email` (str, required, unique)
- `full_name` (str, required)
- `org_id` (str, required)
- `center_id` (str, optional)
- `role_id` (str, required)
- `status` (str, required)
- `created_at` (datetime, defaults to UTC now)
- `updated_at` (datetime, defaults to UTC now, auto-updated on change)
- `last_login` (datetime, optional)
- `created_by` (str, optional)

## Notes

- `create_jerimed_user` pre-checks for existing `email` and `firebase_uid` and raises `409 Conflict` if either already exists; a DB-level `IntegrityError` (race condition) is also caught and converted to the same conflict behavior via `ValueError`.
- Both `PUT` and `PATCH` route to the same `update_user` service method, so there is no distinct full-vs-partial semantic — both apply only the fields explicitly set (`exclude_unset=True`) on `JerimedUserUpdate`.
- Update requests return `404` if the user isn't found and `409` if the update would create a duplicate email or violates a unique constraint (error message content is used to distinguish the two cases: "not found" triggers 404, anything else triggers 409).
- All uncaught exceptions during create/update are converted to `500` with the exception message embedded in the response detail.
- The service layer includes `get_user_by_firebase_uid` and `get_recipients` (list users excluding a given firebase_uid, e.g. for message recipient pickers) methods, but neither is currently wired to a route in `v1/router.py` — they exist as repository/service capability only.
- All endpoints depend on `get_tenant_db`, implying tenant-scoped database sessions (multi-tenant isolation) rather than a shared global database.
