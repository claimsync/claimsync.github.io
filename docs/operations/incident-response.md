# Incident Response

## Severity levels

| Level | Meaning | Example | Response |
|-------|---------|---------|----------|
| SEV1 | Major outage | `/webapi/health` failing across all instances; a tenant's Cloud SQL instance unreachable | Page on-call immediately |
| SEV2 | Partial degradation | One tenant's DB failing to initialize (visible via `/webapi/admin/tenant-db-status`) while others are healthy; migrations blocked for one tenant | Respond during business hours |
| SEV3 | Minor / cosmetic | A single alert misclassified; a slow-but-successful cold start | Track as a normal issue |

## When an incident starts

1. Declare the incident and assign an incident commander.
2. Open a dedicated comms channel.
3. Check the two system endpoints first — they narrow the blast radius quickly:
      - `GET /webapi/health` — overall service health
      - `GET /webapi/admin/tenant-db-status` — per-tenant DB connectivity
4. If a specific tenant is down, check that tenant's Cloud SQL instance directly (GCP console) before assuming an application bug.
5. If migrations are the suspected cause, check server logs for a red `TAMPERED MIGRATION FILE(S) DETECTED` or `NON-STANDARD MIGRATION FILE(S)` block — see [Migration Runner](../guides/mom_backend/MIGRATIONS.md). Migration failures are non-fatal to the server (it still starts), but they block schema changes for the affected tenant.
6. Mitigate first, diagnose second — see [Runbooks](runbooks.md) for the restart/rollback procedures.
7. After resolution, write a blameless postmortem and, if the root cause was a code or migration issue, add it to [Common Mistakes to Avoid](../guides/mom_backend/CLAUDE.md#common-mistakes-to-avoid) or [Runbooks](runbooks.md) so it's faster to recognize next time.
