# Guides

How-to guides and team conventions.

- **[Contributing](contributing.md)** — how to add and edit these docs.
- **[Documentation Style](documentation-style.md)** — writing conventions.

## EHR Pipeline

- **[Monthly Report Flow](ehr_pipeline/monthly_report/monthly_report_flow.md)** — end-to-end walkthrough of `POST /generate-monthly-report`, from request to the tables it writes.

## MOM-Backend

- **[Architecture Guide](mom_backend/CLAUDE.md)** — module structure, request flow, and dependency rules for the FastAPI backend.
- **[Migration Runner](mom_backend/MIGRATIONS.md)** — how startup migrations are discovered, applied, and tamper-checked.
- **[Git Submodule Management](mom_backend/README.md)** — setting up and updating the `medication-optimization-alternatives-search-modules` submodule.
- **[Feature Modules](mom_backend/modules/index.md)** — reference docs for all 21 API modules (endpoints, data model, notes), one page per module.

## Jerimed Frontend
- **[Fall Risk](fall_risk_frontend/fallrisk.md)** -- API reference for the Fall Risk feature's 3 tabs (Triage, Implementation, Care Communication), covering risk scores, medication plan, care team, and education endpoints.
- **[Optimization](optimization_frontend/optimization.md)** -- API reference for the Optimization feature's 3 tabs (Triage, Implementation, Care Communication), covering medication recommendations, triage data, and clinical schedule checklist endpoints.
