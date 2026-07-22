# API Reference

ClaimSync's API surface is served by `mom-backend`, a single FastAPI app hosting
18 clinical analytics modules, plus a separate scheduled endpoint for monthly
reporting. There's no published OpenAPI/Swagger UI yet — this page is the
hand-written reference until one exists.

## Authentication & tenancy

Every request needs two headers, resolved once at the top of the request chain
and never read directly by feature code:

| Header | Purpose |
|---|---|
| `Authorization: Bearer <token>` | Firebase JWT, verified by `verify_firebase_token()`. Identifies the caller. |
| `X-Org-Database` | Selects which tenant's PostgreSQL database the request runs against. |

!!! note
    The caller's RBAC role travels separately, in `X-User-Role-ID`. Role → permission
    mapping lives in `webapi/core/role_permissions.py::ROLE_PERMISSIONS`. See the
    [mom-backend Architecture Guide](../guides/mom_backend/CLAUDE.md#authentication) for the full auth chain.

## Routing pattern

Each feature module is mounted at `/<module>`, with its own `v1` sub-router
underneath:

```
/<module>/v1/<resource>
```

For example, a Medication Appropriateness Index score request:

```
POST /mai/v1/score
```

Adding a `v2` to a module keeps the same prefix pattern (`/<module>/v2/...`) —
see [How to Add a v2 to an Existing Module](../guides/mom_backend/CLAUDE.md#how-to-add-a-v2-to-an-existing-module)
for what's shared vs. duplicated between versions.

## Clinical analytics modules

| Module | Responsibility |
|---|---|
| `mai` | Medication Appropriateness Index |
| `acb` | Anticholinergic Burden |
| `slm` | Sedative Load Model |
| `recommendations` | Maker-checker recommendation flow |
| `fall_risk` | Fall Risk Assessment |
| `reconciliation` | Medication Reconciliation |
| `citations` | Citations |
| `ddi` | Drug-Drug Interactions |
| `hub` | HUB |
| *(+ 9 more)* | See `webapi/api/router.py` for the full list — one `include_router()` per module |

Every module follows the same internal layout (`router.py` → `v1/routers/` →
`v1/services/` → `v1/repositories/` → `v1/models/`) — full detail in the
[Architecture Guide](../guides/mom_backend/CLAUDE.md).

## EHR Pipeline — Monthly Report

A separate scheduled endpoint, triggered by GCP Cloud Scheduler or called
manually, distinct from the per-tenant clinical modules above:

| | |
|---|---|
| **Endpoint** | `POST /generate-monthly-report` |
| **Purpose** | Generates a monthly medication report for a tenant — resolves patient run IDs, queries evaluation scores, and reconciles medications |
| **File** | `services/monthly_report_service.py` |

See the full **[Monthly Report Flow](../guides/ehr_pipeline/monthly_report/monthly_report_flow.md)**
for the request/response schema, the 3-part pipeline, and every table it reads from and writes to.

## Where to go next

- **[mom-backend Architecture Guide](../guides/mom_backend/CLAUDE.md)** — module structure, dependency rules, request-flow trace.
- **[Monthly Report Flow](../guides/ehr_pipeline/monthly_report/monthly_report_flow.md)** — worked example of a full request end to end.
- **[Migration Runner](../guides/mom_backend/MIGRATIONS.md)** — how the database schema each endpoint relies on gets applied.
