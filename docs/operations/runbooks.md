# Runbooks

Step-by-step procedures for recurring `mom-backend` operational tasks.

## Check whether the server is healthy

```bash
curl https://<host>/webapi/health
```

Returns 503 while the search submodule (FAISS/embeddings) is still warming up on a cold start — this is expected briefly after a deploy or Cloud Run scale-up, not necessarily an outage.

## Check per-tenant database status

```bash
curl https://<host>/webapi/admin/tenant-db-status
```

Tenant engines initialize in a background thread at startup so the server accepts requests immediately; this endpoint shows which tenants have finished initializing and which (if any) failed to connect.

## Restart a deployed environment (GCP VM path)

1. SSH to the GCP VM
2. `sudo supervisorctl status` — confirm which service is affected (`flask_app_mom_dev` / `_staging` / `_prod`)
3. `sudo supervisorctl restart <service>`
4. Re-check `/webapi/health` on that environment's port (9001/9002/9003)

## Roll back a bad deploy (GCP VM path)

1. `cd` into the environment's app directory (see the branch → directory table in [Deployment](deployment.md))
2. `git log` to find the last-known-good commit
3. `git reset --hard <good-commit>`
4. `sudo supervisorctl restart <service>`

## Investigate a blocked migration

If a tenant stops applying new migrations, check the server logs for a red `TAMPERED MIGRATION FILE(S) DETECTED` block — this means an already-applied migration file was edited after the fact, which the runner refuses to proceed past for that tenant. See [Migration Runner](../guides/mom_backend/MIGRATIONS.md#5-_verify_checksums-file-integrity-check) for the exact fix (revert the file, or add a new migration instead of editing).

## Rotate a per-tenant DB credential

1. Update the credential in the source of truth (GCP Secret Manager, or the `{TENANT_KEY}_DB_PASSWORD` env var for non-IAM tenants)
2. Redeploy or restart the affected environment so `webapi/db/session.py` picks up the new credential when it re-initializes tenant engines
3. Confirm via `/webapi/admin/tenant-db-status` that the tenant reconnected successfully

## Add a new IAM-authenticated tenant

Only needed for Cloud SQL IAM tenants (currently `longevity`). See the [mom-backend Architecture Guide](../guides/mom_backend/CLAUDE.md#adding-a-new-iam-tenant) for the full steps (`TENANT_DB_NAMES`, `TENANT_IAM_CONFIGS`, required env vars).
