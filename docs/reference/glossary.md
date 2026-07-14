# Glossary

ClaimSync- and mom-backend-specific terms and acronyms.

| Term | Definition |
|------|------------|
| **ACB** | Anticholinergic (Cognitive) Burden — a score summing a patient's exposure to anticholinergic medications. See [ACB module](../guides/mom_backend/modules/acb.md). |
| **AHFS** | American Hospital Formulary Service — the source of drug monograph sections shown in citation lookups. See [Citations module](../guides/mom_backend/modules/citations.md). |
| **CMR** | Comprehensive Medication Review — the Medicare Part D MTM review workflow. See [CMR Reviews module](../guides/mom_backend/modules/cmr-reviews.md). |
| **DDI** | Drug-Drug Interaction — pairwise interaction scoring between a patient's medications. See [DDI module](../guides/mom_backend/modules/ddi.md). |
| **FDB** | First Databank — an external reference database of drug/formulary data (interactions, geriatric risk flags, AHFS monographs). |
| **GCN / GCN Seqno** | Generic Code Number (Sequence Number) — First Databank's identifier for a specific drug formulation, used throughout the backend to identify medications independent of brand name. |
| **HEDIS** | Healthcare Effectiveness Data and Information Set — the standard behind many of the [Quality Measures](../guides/mom_backend/modules/quality-measures.md) (CBP, GSD, KED, SUPD, etc.). |
| **IDT** | Interdisciplinary Team — the care-planning team whose decisions/actions are tracked in [Fall Risk Assessment](../guides/mom_backend/modules/fall-risk.md). |
| **IPSD** | Index Prescription Start Date — the anchor date PDC-style quality measures use to define their measurement window. |
| **MAI** | Medication Appropriateness Index — a per-medication, multi-criterion appropriateness rating. See [MAI module](../guides/mom_backend/modules/mai.md). |
| **MME** | Morphine Milligram Equivalent — the normalized opioid dosage unit used to assess opioid risk. See [Opioid module](../guides/mom_backend/modules/opioid.md). |
| **MTM** | Medication Therapy Management — the CMS program behind the MTM quality measure and the CMR review requirement. |
| **PACE / PaceSemi** | The geriatric care programs `mom-backend` serves as tenants. |
| **PDC** | Proportion of Days Covered — a medication-adherence percentage used by several [Quality Measures](../guides/mom_backend/modules/quality-measures.md) (PDC-DIAB, PDC-RASA, PDC-Statin). |
| **PRN** | *Pro re nata* — "as needed" medication dosing, tracked as a flag on medication records across most clinical modules. |
| **Prescribing cascade** | A chain where a new medication is prescribed to treat a side effect of another medication, rather than addressing the underlying cause. See [Cascade module](../guides/mom_backend/modules/cascade.md). |
| **run_id / Evaluation run** | A batch scoring pass for a patient (`evaluation_runs` table). Most clinical modules read the "latest completed" run, and many additionally require it to be `validated`. |
| **SLM** | Sedative Load Model — a per-patient sedative medication burden score. See [SLM module](../guides/mom_backend/modules/slm.md). |
| **Tenant** | One customer organization's isolated database. Selected per-request via the `X-Org-Database` header; see [Architecture Overview](../architecture/overview.md#multi-tenancy). |
| **TDD** | Total Daily Dose — an aggregated medication dosing figure used in the [Monthly Report Flow](../guides/ehr_pipeline/monthly_report/monthly_report_flow.md). |
| **Validated** | A flag on an evaluation run indicating a reviewer has signed off on it. Non-privileged roles are usually restricted to validated-only data; see `get_validated_filter` in the [Architecture Guide](../guides/mom_backend/CLAUDE.md). |
