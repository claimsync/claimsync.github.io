# Deployment

`mom-backend` has two independent deployment paths in use.

## Path 1 — GCP VM via supervisor (primary)

A push to `develop`, `staging`, or `production` triggers a GitHub Actions workflow that SSHes into a GCP VM, hard-resets the branch, updates the submodule, and restarts the matching `supervisor` service.

| Branch | Port | Supervisor service | App directory | Submodule branch |
|---|---|---|---|---|
| `develop` | 9001 | `flask_app_mom_dev` | `/opt/flask-apps/mom-backend-dev` | `develop` |
| `staging` | 9002 | `flask_app_mom_staging` | `/opt/flask-apps/mom-backend-staging` | `main` |
| `production` | 9003 | `flask_app_mom_prod` | `/opt/flask-apps/mom-backend-production` | `main` |

Deploy steps (automated):

1. `git fetch --all && git checkout <branch> && git reset --hard origin/<branch> && git pull --rebase`
2. `git submodule update --init --recursive`, then check out and pull the submodule's own branch (per the table above)
3. If the submodule pointer changed, auto-commit that bump to the main repo
4. `pip install -r requirements.txt` inside a conda env (`mom_backend_py39`)
5. `sudo supervisorctl reread && sudo supervisorctl update && sudo supervisorctl restart <service>`

There is no separate approval gate — a push to one of the three branches deploys directly. **Rollback** is a `git reset --hard` to a prior commit on that branch (or a revert-and-push) followed by the same supervisor restart.

!!! note
    Every push runs migrations automatically on server startup (see [Migration Runner](../guides/mom_backend/MIGRATIONS.md)) — there's no separate "run migrations" deploy step.

## Path 2 — Cloud Run (container image)

Built from the repo's `Dockerfile` via `cloudbuild.yaml`:

1. Init the private submodule using a `github-pat` secret
2. Build a multi-stage image — a `gcc`-equipped builder stage compiles `psycopg2`, then a lean `python:3.11-slim` runtime stage copies only the installed packages plus `app.py`, `configs/`, `webapi/`, and `core_business_logic/src` + `core_business_logic/data`
3. The image pre-downloads the MedEmbed HuggingFace model (`abhinand/MedEmbed-large-v0.1`) at **build time** so it doesn't block cold starts
4. Push to Artifact Registry, then `gcloud run deploy` with `--min-instances=1` and a startup probe against `/webapi/health`

At container start, `startup.sh` (the image's `CMD`) does three things before launching uvicorn on port 8080:

1. Downloads prebuilt search databases from GCS if `SEARCH_DB_GCS_PATH` is set
2. Downloads guideline PDFs from the `guideline_papers` GCS bucket
3. Pre-warms heavy imports (`torch`, `faiss`, `transformers`) so the first real request isn't the one paying that cost

The Cloud Run deploy step only updates the image — no env var flags are passed, so existing Cloud Run environment variables are preserved across deploys.

## Environments

| Environment | Path | Port |
|---|---|---|
| Local dev | `python app.py` | 9001 |
| `develop` | GCP VM / supervisor | 9001 |
| `staging` | GCP VM / supervisor | 9002 |
| `production` | GCP VM / supervisor | 9003 |
| Cloud Run | container image | 8080 |
