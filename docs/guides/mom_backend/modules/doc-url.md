# Doc URL

Provides an endpoint that resolves a document, client, and patient identifier into an accessible document URL by delegating to an external "doc-linker" Cloud Run service, authenticating the call with a Google-issued ID token.

## Endpoints

| Method | Path | Purpose |
|---|---|---|
| POST | `/webapi/api/v1/get_doc_url` | Retrieve a document URL for a given `doc_id`, `client_id`, and `patient_id` |

## Data model

- `DocUrlRequest` (request body): `doc_id: str`, `client_id: str`, `patient_id: str`
- Response (JSON, not a typed schema): `success: bool`; `data` — raw JSON (or `{"url": ...}`) returned by the doc-linker API on success; `error`/`details` on failure; `request_params` echoes back `doc_id`, `client_id`, `patient_id`

## Notes

- Calls an external "doc linker" service (`DOC_LINKER_API_URL`, default `https://doc-linker-664478834178.us-central1.run.app/doc-link`) rather than generating a signed URL locally.
- Authenticates to that service via a Google-issued ID token: uses `GCP_SERVICE_ACCOUNT_JSON` service-account credentials if set, otherwise falls back to ambient credentials via `google.oauth2.id_token.fetch_id_token`.
- `client_id` is remapped before being sent upstream via a hardcoded map (`claimsync` → `Sandbox`, `real_sandbox` → `RealSandbox`); other values pass through unchanged.
- Request timeout to the upstream API is configurable via `DOC_LINKER_API_TIMEOUT` (default 30s); timeouts and network errors are caught and surfaced as failures rather than raising.
- Any upstream failure (non-200 status, timeout, network error) results in a 500 response with `success: false` and details of the failure; unexpected exceptions in the route handler are also caught and returned as a generic 500 "Internal server error" response.
