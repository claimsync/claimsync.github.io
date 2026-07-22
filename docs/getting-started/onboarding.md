# Onboarding

A checklist for new team members working on ClaimSync's backend, `mom-backend`.

## First day

- [ ] Get access to the `claimsync` GitHub org, including `mom-backend` and its submodule `medication-optimization-alternatives-search-modules`
- [ ] Get GCP access (Cloud SQL, Cloud Storage, Secret Manager) if you'll touch tenant databases or deploys
- [ ] Get a Firebase project role if you'll test authenticated endpoints
- [ ] Join the team communication channels
- [ ] Clone `mom-backend` **with submodules** — see [Local Setup](local-setup.md)

## First week

- [ ] Read the [Architecture Overview](../architecture/overview.md) — request flow, module layout, dependency rules, multi-tenancy
- [ ] Read the [mom-backend Architecture Guide](../guides/mom_backend/CLAUDE.md) for the full conventions (adding a module, adding a `v2`, naming rules)
- [ ] Complete [Local Setup](local-setup.md) and get the server running against a tenant DB
- [ ] Skim the [Feature Modules](../guides/mom_backend/modules/index.md) index to see what's already built — avoid duplicating an existing module
- [ ] Read [Migration Runner](../guides/mom_backend/MIGRATIONS.md) before writing your first schema change
- [ ] Pick a starter task — a small addition to an existing module is the fastest way to see the request → service → repository → DB chain in practice

!!! tip
    `mai/` is the only module with a `v2` — read its two versions side by side (in the [MAI module doc](../guides/mom_backend/modules/mai.md)) if you need to add a `v2` to your own module later.
