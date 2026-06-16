# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

---

## Project Overview

**ClaimsSync** is a multi-tenant healthcare data management platform.
The backend is a FastAPI application that powers clinical analytics modules
for PACE and PaceSemi care programs.

- **Language:** Python 3.11+
- **Framework:** FastAPI (sync endpoints — plain `def`, not `async def`)
- **ORM:** SQLModel / SQLAlchemy 2.x **sync** (`sqlmodel.Session`) — no AsyncSession
- **Auth:** Firebase Authentication + JWT (verified in `webapi/core/security.py`)
- **Infrastructure:** GCP — Cloud SQL (PostgreSQL), Cloud Storage (GCS)
- **Multi-tenancy:** Per-request tenant DB resolved from `X-Org-Database` header via `webapi/core/dependencies.py`

---

## Architecture — Feature-based Domain Structure

Code is organized by **clinical feature module**, not by technical layer.
Each module is self-contained and independently versioned.

Real root is `mom-backend/`. All application code lives under `webapi/`.

```
mom-backend/
├── app.py                     # Process entry point — launches uvicorn webapi.main:app on port 9001
├── core_business_logic/src/   # Clinical scoring engines (MAI, ACB, SLM, DDI, fall risk)
│                              # Injected into sys.path at startup; imports as bare module names
└── webapi/
    ├── main.py                # FastAPI app factory, lifespan, middleware, health endpoints
    ├── api/
    │   └── router.py          # Pure wiring — one include_router() per feature module (18 total)
    ├── core/                  # Cross-cutting — never imports from feature modules
    │   ├── config.py          # Pydantic BaseSettings (env vars)
    │   ├── security.py        # Firebase JWT verification → request.state.firebase_user
    │   ├── dependencies.py    # get_tenant_db(), get_current_user(), get_user_role()
    │   └── role_permissions.py# ROLE_PERMISSIONS dict for RBAC
    ├── middleware/
    │   ├── tenant_context.py  # Reads X-Org-Database → request.state.tenant_db_name
    │   └── internal_auth.py   # Service-to-service auth (toggled via INTERNAL_AUTH_ENABLED)
    ├── db/
    │   ├── session.py         # Sync engine factory, _tenant_engines cache, init_tenant_engines()
    │   ├── tenant_config.py   # TENANT_DB_NAMES + TENANT_IAM_CONFIGS
    │   └── base.py            # SQLAlchemy declarative base
    ├── migrations/            # Raw SQL migration files (no Alembic) — apply via order.txt
    ├── shared/                # Cross-feature schemas and utilities
    │
    ├── mai/                   # Medication Appropriateness Index  ┐
    ├── acb/                   # Anticholinergic Burden            │
    ├── slm/                   # SLM                               │
    ├── recommendations/       # Maker-checker recommendation flow │  all follow
    ├── fall_risk/             # Fall Risk Assessment              │  the same
    ├── reconciliation/        # Medication Reconciliation         │  module
    ├── citations/             # Citations                         │  layout
    ├── ddi/                   # Drug-Drug Interactions            │  below
    ├── hub/                   # HUB                               │
    └── ...                    # + 9 more modules                  ┘

Each feature module layout:
    <module>/
    ├── router.py          # Mounts v1_router at prefix="/<module>"
    └── v1/
        ├── router.py      # Wires sub-routers — pure wiring, no endpoints
        ├── routers/       # Endpoint handlers (one file per resource group)
        ├── services/      # Business logic
        ├── repositories/  # All DB queries
        ├── models/        # SQLModel ORM table classes
        └── schemas/       # Pydantic request/response models
```

---

## Request Flow

Every API call follows this exact chain — nothing skips a layer:

```
POST /webapi/api/v1/mai/v1/score
  → TenantContextMiddleware   (X-Org-Database → request.state.tenant_db_name)
  → api/router.py             (routes to mai_router)
  → mai/router.py             (routes to v1_router)
  → mai/v1/router.py          (routes to endpoint in routers/)
  → mai/v1/routers/*.py       (handler — Depends(get_tenant_db) → Session)
  → mai/v1/services/          (business logic)
  → mai/v1/repositories/      (DB query via SQLModel ORM)
  → PostgreSQL (tenant DB)
```

---

## Dependency Rules (Critical)

```
feature module  →  shared/  →  core/  →  db/
```

| Layer | Can import from | NEVER imports from |
|---|---|---|
| Feature module (e.g. mai/) | shared/, core/, db/ | Other feature modules |
| shared/ | core/, db/ | Any feature module |
| core/ | db/ (config only) | Features, shared/ |
| api/router.py | Feature routers only | Services, repos, models |

**If two feature modules need to share code — move it to `shared/`, not a direct import.**

---

## How to Add a New Clinical Module

1. `mkdir -p webapi/new_module/v1/{routers,schemas,services,repositories,models}`
2. Create `webapi/new_module/router.py`:
   ```python
   from fastapi import APIRouter
   from webapi.new_module.v1.router import router as v1_router
   router = APIRouter(prefix="/new_module", tags=["New Module"])
   router.include_router(v1_router, prefix="/v1")
   ```
3. Add one line to `webapi/api/router.py`:
   ```python
   from webapi.new_module.router import router as new_module_router
   api_router.include_router(new_module_router)
   ```

No other existing files need to change.

---

## How to Add a v2 to an Existing Module

**ORM models are never versioned.** They map to real DB tables — duplicating them creates two classes fighting over the same table. Only the API surface (schemas, services, routers) changes between versions.

What to copy vs. what to share:

| Layer | Rule |
|---|---|
| `models/` | **Shared** — `v2` imports from `v1/models/`, never duplicates |
| `repositories/` | **Shared or extended** — create `v2/repositories/` only if queries change; otherwise import from `v1` |
| `services/` | **New in v2** — business logic typically changes between versions |
| `schemas/` | **New in v2** — request/response shapes are what versioning is for |
| `routers/` | **New in v2** — new endpoint handlers calling new services |

Steps:

1. Create only what changes:
   ```
   webapi/mai/v2/
   ├── router.py        # wiring
   ├── routers/         # new endpoint handlers
   ├── services/        # new business logic
   └── schemas/         # new request/response shapes
   # NO models/ — import from v1
   ```

2. In `v2` services/repositories, import the shared models from `v1`:
   ```python
   from webapi.mai.v1.models.mai_score import MAIScore  # same table, no duplicate
   ```

3. Wire v2 in `webapi/mai/router.py`:
   ```python
   from webapi.mai.v2.router import router as v2_router
   router.include_router(v2_router, prefix="/v2")
   ```

4. `v1/` remains completely untouched. `api/router.py` is not touched.

---

## Key Patterns

### Dependency injection
All shared resources (DB session, current user, tenant) are injected via FastAPI `Depends()`.
Never instantiate a DB session or decode a JWT inside an endpoint handler directly.

```python
# Endpoints are plain def (not async def) — the ORM is synchronous
@router.get("/score/{patient_id}")
def get_score(
    patient_id: str,
    db: Session = Depends(get_tenant_db),
) -> ScoreResponse:
    service = MAIService(db)
    return service.compute_score(patient_id)
```

### Repository pattern
Feature repositories are plain classes that receive a `Session` and run queries.
They are not subclasses of a shared BaseRepository.

```python
class MAIRepository:
    def __init__(self, session: Session):
        self.session = session

    def get_scores(self, patient_id: str) -> list[MAIScore]:
        return self.session.exec(select(MAIScore).where(...)).all()
```

### Service layer
Services contain all business logic. They call repositories, never the ORM directly.
Endpoints call services, never repositories directly.

### Multi-tenancy
`TenantContextMiddleware` reads `X-Org-Database` from the request header and stores it on
`request.state.tenant_db_name`. `get_tenant_db()` then resolves the correct engine from a
global cache. Never read `X-Org-Database` directly inside a service or repository.

---

## Database

- Primary: PostgreSQL via Cloud SQL (multi-tenant — one engine per tenant, cached globally)
- ORM: SQLModel / SQLAlchemy 2.x **sync** — sessions are `sqlmodel.Session`, not `AsyncSession`
- Migrations: **raw SQL files** in `webapi/migrations/` — apply manually in the order listed in `webapi/migrations/order.txt`. There is no Alembic.
- Feature-specific ORM models live in `<module>/v1/models/`

Common NULL gotcha: `ps_center_id` can be NULL for some patient records.
Always filter with `IS NOT NULL` when center context is required.

---

## Cloud SQL IAM Tenants (Longevity + future IAM tenants)

Some tenants authenticate via **Cloud SQL IAM** instead of a password.
`longevity` is the first. The rules below apply to every tenant listed in
`db/tenant_config.py::TENANT_IAM_CONFIGS`.

### How it works

| Layer | File | What it does |
|---|---|---|
| Config | `db/tenant_config.py` | `TENANT_IAM_CONFIGS` maps tenant key → Cloud SQL instance name. `_get_connector()` lazily creates a single `google.cloud.sql.connector.Connector` (shared across IAM tenants). `_make_iam_creator()` returns a pg8000 `creator` callable. |
| Engine init | `db/session.py::_init_one_tenant` | Detects IAM tenant, calls `_patch_pg8000_once()` + `_attach_uuid_coercion()`, then creates a `postgresql+pg8000://` engine via `create_engine(creator=...)`. |
| UUID fix | `db/session.py::_attach_uuid_coercion` | `before_cursor_execute` listener that converts bare `str` UUIDs → `uuid.UUID` objects so pg8000 sends OID 2950, not varchar. Without this PostgreSQL throws `operator does not exist: uuid = character varying`. |
| Bind cast fix | `db/session.py::_patch_pg8000_once` | Sets `_PGString.render_bind_cast = False` once per process to suppress unnecessary SQLAlchemy cast injection for pg8000. |

### Required environment variables

IAM tenants do **not** use `{TENANT}_DB_HOST` or `{TENANT}_DB_PASSWORD`.

```
LONGEVITY_DB_NAME=longevity_db           # database name on the Cloud SQL instance
LONGEVITY_DB_USER=svc-account@project.iam.gserviceaccount.com
LONGEVITY_DB_CREDENTIALS_SECRET=projects/.../secrets/.../versions/latest  # optional: Secret Manager path to a service-account JSON key
```

If `LONGEVITY_DB_CREDENTIALS_SECRET` is absent, the Connector uses **Application Default Credentials** (ADC) — suitable for Cloud Run with a bound service account.

### Adding a new IAM tenant

1. Add to `TENANT_DB_NAMES` in `db/tenant_config.py`:
   ```python
   "new_tenant": "new_tenant_db",
   ```
2. Add to `TENANT_IAM_CONFIGS`:
   ```python
   "new_tenant": "gcp-project:region:instance-name",
   ```
3. Set env vars: `NEW_TENANT_DB_NAME`, `NEW_TENANT_DB_USER`.
4. Optionally set `NEW_TENANT_DB_CREDENTIALS_SECRET` if ADC is not available.
5. No changes to `session.py` — the `_init_one_tenant` branch is generic.

### Rules for feature code touching IAM tenants

- **Never bypass `get_tenant_db`** in endpoints — the UUID coercion is attached to the engine, not the session; it fires automatically.
- **Never construct a raw psycopg2 connection** for an IAM tenant — pg8000 is required; psycopg2 cannot authenticate via Cloud SQL IAM.
- **Never use positional `%s` params** in raw SQL via `session.exec(text(...))` on an IAM tenant — use SQLAlchemy named params (`:param_name`) or ORM expressions instead.
- **UUID columns must not be passed as plain strings** in ORM filter expressions (e.g. `Model.id == some_str`) — use `uuid.UUID(some_str)` or let the coercion listener handle it via `text()` queries.

---

## Authentication

Firebase JWT via `Authorization: Bearer <token>`. Verified by `verify_firebase_token()` in
`webapi/core/security.py`, which stores the decoded payload on `request.state.firebase_user`.

`get_current_user(request)` in `dependencies.py` returns `(user_id, email, name)` — first from the
decoded Firebase token on `request.state`, then falling back to `X-User-ID` / `X-User-Email` /
`X-User-Name` headers.

The `X-User-Role-ID` header carries the caller's RBAC role. Read it via `get_user_role(request)`.
Role permissions are declared in `webapi/core/role_permissions.py::ROLE_PERMISSIONS`.
Use `get_validated_filter(request)` to get the `AND validated = true` SQL clause for non-privileged users.

---

## Environment Variables

Declared in `webapi/core/config.py` as a Pydantic `BaseSettings` model.

Key variables:
- `{TENANT_KEY}_DB_HOST`, `{TENANT_KEY}_DB_NAME`, `{TENANT_KEY}_DB_USER`, `{TENANT_KEY}_DB_PASSWORD` — per-tenant DB credentials (e.g. `SANDBOX_DB_HOST`). Tenants are listed in `webapi/db/tenant_config.py::TENANT_DB_NAMES`.
- `DEFAULT_TENANT_DB` — fallback tenant when `X-Org-Database` header is absent
- `FIREBASE_PROJECT_ID` — for token verification
- `GCS_BUCKET_NAME` — Cloud Storage bucket
- `INTERNAL_AUTH_ENABLED` — toggle service-to-service auth middleware
- `ENV` — `development | staging | production`

IAM tenant additional vars (e.g. `longevity`): `LONGEVITY_DB_NAME`, `LONGEVITY_DB_USER`, optionally `LONGEVITY_DB_CREDENTIALS_SECRET`.

---

## Testing

```bash
pytest tests/ -v
pytest tests/path/to/test_file.py::test_name -v   # single test
```

Clinical scoring unit tests live in `core_business_logic/src/` and can be run directly:
```bash
cd core_business_logic/src && python test_driver.py
```

Use `pytest-asyncio` for any async test helpers. Use `httpx.AsyncClient` with the FastAPI `app` object directly for integration tests — no live server needed.

---

## Common Mistakes to Avoid

- **Never import between feature modules.** If you need something from `acb/` inside
  `mai/`, it belongs in `shared/`.
- **Never put endpoints in `api/router.py`.** It is wiring-only.
- **Never skip the service layer.** Endpoints call services, services call repositories.
  Endpoints never call repositories directly.
- **Never use `round()` where `math.ceil()` is needed for dose calculations.** The
  team made an explicit decision to use `round()` for total daily dose formatting —
  do not change this without a discussion.
- **Never call synchronous `requests` inside code that runs on the async event loop.** Use `httpx.AsyncClient` for async contexts.
- **Always filter NULL `ps_center_id`** when querying patients scoped to a center.

---

## File Naming Conventions

| Type | Convention | Example |
|---|---|---|
| Service files | `{module}_service.py` | `mai_service.py` |
| Repository files | `{module}_repo.py` | `mai_repo.py` |
| Schema files | `{noun}.py` | `request.py`, `response.py` |
| ORM model files | `{entity}.py` | `patient.py`, `evaluation_run.py` |
| Test files | `test_{module}.py` | `test_mai_router.py` |

---

## Useful Commands

```bash
# Run dev server (from mom-backend/)
python app.py
# or:
uvicorn webapi.main:app --host 0.0.0.0 --port 9001 --reload --reload-dirs webapi,core_business_logic

# Run tests
pytest tests/ -v
pytest tests/path/to/test_file.py::test_name -v

# Apply a migration manually
psql -h <host> -U <user> -d <db> -f webapi/migrations/<migration>.sql

# Connect via IAP tunnel (use personal gcloud account, not service account)
gcloud compute start-iap-tunnel ...
```

---

*Keep this file updated when the architecture changes. It is read by AI tools on every session.*
