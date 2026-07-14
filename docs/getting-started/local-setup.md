# Local Setup

Setting up `mom-backend` for local development.

## Prerequisites

- Python 3.11+
- Git (with SSH access to the `claimsync` org)
- Access to a tenant PostgreSQL database (or a local Postgres instance for `sandbox_staging_db`)

## 1. Clone with the submodule

The clinical scoring engines (MAI, ACB, SLM, DDI, fall risk) live in a separate Git submodule, `medication-optimization-alternatives-search-modules`, mounted at `core_business_logic/`. Without it, DDI / medication-alternatives and FAISS search won't import.

```bash
git clone --recursive git@github.com:claimsync/mom-backend.git
# or, if already cloned without --recursive:
git submodule update --init --recursive
```

See [Git Submodule Management](../guides/mom_backend/README.md) for updating the submodule later.

## 2. Create a virtualenv and install dependencies

```bash
python3.11 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
pip install -r requirements.txt
```

The first install is large — it pulls `torch`, `transformers`, and `faiss-cpu`. The MedEmbed embedding model (`abhinand/MedEmbed-large-v0.1`) downloads on first use.

## 3. Configure environment

Settings load via Pydantic (`webapi/core/config.py`) and per-tenant DB config (`webapi/db/tenant_config.py`). Create a `.env` file in the repo root — coordinate with the team for real secrets (Firebase, GCP Secret Manager, per-tenant DB credentials). Common variables:

```bash
# App
DEBUG=true
CORS_ORIGINS=*

# Default / fallback PostgreSQL
PG_HOSPITAL_HOST=localhost
PG_HOSPITAL_PORT=5432
PG_HOSPITAL_DATABASE=sandbox_staging_db
PG_HOSPITAL_USER=postgres
PG_HOSPITAL_PASSWORD=...

# Per-tenant DBs (one set per active tenant key, e.g. PACE_*, PACESEMI_*, SANDBOX_*)
PACE_DB_HOST=...
PACE_DB_NAME=pace_north_db
PACE_DB_USER=...
PACE_DB_PASSWORD=...

# Tenant routing
DEFAULT_TENANT_DB=sandbox_staging_db

# Auth / middleware
INTERNAL_AUTH_ENABLED=false   # false in dev, true in staging/prod
```

## 4. Run the server

```bash
python app.py
# serves on http://0.0.0.0:9001 with autoreload
```

- API docs: `http://localhost:9001/webapi/docs`
- Health check: `http://localhost:9001/webapi/health`
- Per-tenant DB init status: `http://localhost:9001/webapi/admin/tenant-db-status`

Migrations run automatically per tenant at startup — see [Migration Runner](../guides/mom_backend/MIGRATIONS.md). No manual `psql` step needed.

## 5. Call an endpoint

All feature endpoints require an auth token and (typically) an `X-Org-Database` header identifying the tenant:

```bash
curl http://localhost:9001/webapi/api/v1/<module>/v1/<endpoint> \
  -H "Authorization: Bearer <firebase-or-google-id-token>" \
  -H "X-Org-Database: sandbox_staging_db"
```

## Run the docs site locally

```bash
pip install -r requirements.txt   # from claimsync.github.io/
mkdocs serve                      # http://127.0.0.1:8000
```
