# Drug-Drug Interactions (DDI)

This module provides two related capabilities: a drug-drug interaction (DDI) scoring matrix for a list of drugs, and a medication-alternatives recommender that ranks substitute drugs for a given "seed" drug against a patient's clinical profile (age, renal/hepatic function, diagnoses, allergies, current medications). "Medication alternatives" here means a scored/ranked list of substitute drugs produced by a multi-factor clinical ranker (safety, geriatric appropriateness, indication match, DDI, source-boosting), not a simple lookup table.

## Endpoints

| Method | Path | Purpose |
|---|---|---|
| POST | `/webapi/api/v1/ddi/interaction_matrix` | Returns a pairwise interaction matrix and per-drug interaction scores for a list of drugs |
| POST | `/webapi/api/v1/medication-alternatives` | Ranks alternative medications for a seed drug given a patient's clinical context |
| GET | `/webapi/api/v1/medication-alternatives/health` | Health check for the medication-alternatives ranking system |
| POST | `/webapi/api/v1/medication-alternatives/initialize` | Force (re)initialization of the medication-alternatives ranking system |

## Data model

**DDIInteractionMatrixRequest** (`/ddi/interaction_matrix`)

- `drugs`: list of `{gcn_seqno: int, drug_name: str}` — must be non-empty
- `preserve_order`: bool, default `false` — if false, response drugs/matrix are re-sorted by total interaction score descending

Response: `{drugs: [{name, gcn_seqno, total_score, display_order}], interactionMatrix: number[][], interactionDescriptions: {...}, metadata: {...}}`

**MedicationAlternativesRequest** (`/medication-alternatives`)

- `seed_drug`: `{drug_name: str, gcn: str}`
- `patient`: `{patient_id, age, sex?, egfr?, hepatic_function?, total_acb?, total_slm?, falls_last_year?, diagnoses?, allergies?, current_medications?}`
- `config_overrides`: optional dict, restricted to an allowlist (`ranker.same_route`, `ranker.same_dosage_form`, `ranker.max_etc_level` [1-100], `ranker.extended_release_only`, `ranker.min_shared_indications` [0-10])

Response: the ranker's result merged with `{success, patient_id, alternatives_count, config_overrides_applied, cached}`.

## Notes

- **Mount-point clarification**: `webapi/ddi/router.py` mounts its v1 router under `/ddi`, so the interaction-matrix endpoint lands at `/ddi/interaction_matrix`. However, `webapi/api/router.py` imports `med_alt_router` **directly** from `webapi.ddi.v1.medication_alternatives_router`, bypassing `webapi/ddi/router.py`'s `/ddi` prefix entirely. Since that router declares its own `APIRouter(prefix="/medication-alternatives")`, it mounts as a **separate, top-level path** (`/webapi/api/v1/medication-alternatives`), sibling to `/ddi` rather than nested under it — despite living in the `ddi/v1` package.
- Both the DDI matrix service and the medication-alternatives endpoint first try an external `jerimed_service` (via `post_jerimed`, only when env var `JERIMED_EXTERNAL_SERVICE_URL` is set) and fall back to direct in-process logic on any exception/unreachability.
- Direct-path DDI scoring uses `core_business_logic`'s `DrugInteractionManager` against an FDB (First Databank) reference database; raw severities (1/2/3) are mapped to points (1/5/25), summed per drug, and used to reorder the matrix.
- Direct-path medication alternatives is heavyweight: a background thread lazily builds a shared `AlternativeRanker` (TXA, ETC, Indication, Safety, Geriatric, Clinical/DDI, optional Semantic processors) backed by Postgres/FDB, an optional HuggingFace-embedding-based drug matcher, and an LLM-driven SLM classifier — `/health` and `/initialize` exist to check/force this async startup, returning 503 while not ready.
- Successful `/medication-alternatives` results are cached in-memory for 1 hour keyed by a hash of the seed GCN plus patient attributes; the cache is skipped whenever `config_overrides` are supplied.
- Error handling differs by endpoint: `/ddi/interaction_matrix` raises `HTTPException(500, detail=str(e))` (leaks exception text); `/medication-alternatives` catches exceptions and returns a generic 500 JSON response without exception detail, and returns 400 for invalid config overrides.
