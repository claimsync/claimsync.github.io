# Medication Alternatives Module — Complete Reference

**Author:** Keertiraj DJ

`POST /medication-alternatives`

Given a seed drug (e.g. gabapentin) and a patient profile (age, diagnoses, current medications, renal/hepatic status), this module finds every plausible replacement drug, evaluates each one for safety and geriatric appropriateness specific to that patient, and returns a ranked, scored list bucketed into Recommended / Consider / Use With Caution.

This is a deep internals reference for the medication-alternatives half of the [DDI module](ddi.md) — start there for the endpoint contract and the shorter overview.

---

## Repository Layout

```
mom-backend/
├── webapi/ddi/v1/
│   ├── medication_alternatives_router.py      # FastAPI router, result cache, request validation
│   └── services/
│       └── medication_alternatives_service.py # All orchestration, monkey-patches, env switches
├── configs/
│   ├── med_alternative.yaml                    # Weights, penalties, processor settings
│   ├── drug_matcher_config.yaml                # LLM model + tokens for drug disambiguation
│   └── general_model_config.yaml               # General LLM config (SLM CoT — currently hardcoded off)
├── core_business_logic/                        # git submodule
│   └── src/
│       ├── drug_alternative/
│       │   ├── rankers/alternative_ranker.py   # AlternativeRanker — top-level orchestrator
│       │   └── processors/
│       │       ├── geriatric_processor.py      # ACB + SLM + Beers
│       │       ├── safety_processor.py         # Contraindications, BBW, side effects
│       │       ├── clinical_data_processor.py  # DDI, renal/hepatic, REMS
│       │       ├── indication_processor.py     # Jaccard similarity on ICD-10 codes
│       │       ├── txa_processor.py            # Therapeutic Exchange Alternatives
│       │       └── etc_processor.py            # Equivalent Therapeutic Class
│       ├── data/fetcher.py                     # AlternativeDataFetcher — all DB queries
│       ├── slm/slm_matcher.py                  # SLMLookupSystem — 4-tier GCN→sedative score
│       └── drug_search/
│           ├── drug_matcher.py                 # FDBDrugMatcherWithLLM — name→GCN matching
│           └── fuzzy_drug_search.py            # FAISSPharmaceuticalSearchEnhanced (heavy)
└── data/
    ├── acb/enriched_acb_dictionary.json        # Anticholinergic burden scores keyed by GCN
    └── slm/gcn_enriched_drugs.json             # Sedative load scores keyed by GCN (trusted source, read-only)
```

---

## Initialization (happens once at startup, in a daemon thread)

`MedicationAlternativesManager.initialize()` blocks the readiness gate (`/health`) until complete:

1. Load and cache `configs/med_alternative.yaml`
2. Build `_TunedPostgreSQLManager` (main FDB pool, min=5 max=20, keepalives enabled) — this is `db_manager`
3. Build `AlternativeDataFetcher(db_manager)` and wrap its 13 methods with `_FetchMemo` (TTL cache)
4. Initialize 6 processors (TXA, ETC, Indication, Safety, Geriatric, Clinical) — all receive the same shared fetcher instance
5. Initialize `GeriatricProcessor`:
   - Load ACB dictionary from JSON
   - Build `FDBDrugMatcherWithLLM` using `configs/drug_matcher_config.yaml` (creates its own PostgreSQL pool, min=2 max=10, no keepalives)
   - Build `SLMLookupSystem(drug_matcher, slm_data_path, slm_classifier)` then immediately override `slm_system.db = db_manager` (replaces drug_matcher's pool with the always-warm main pool — fixes cold-start Tier 2 failures)
   - Call `_apply_disable_vector_search(ranker)` — sets `drug_matcher.vector_search = None` unless `MEDALT_VECTOR_SEARCH_ENABLED=true`
6. Build `AlternativeRanker` with all processors
7. Apply monkey-patches to the ranker: `_apply_parallel_evaluate_all`, `_apply_parallel_enrich_attach`
8. Ready gate opens — requests unblocked

---

## Request Flow (6 steps)

### Step 1 — Enrich the seed drug (~2s)

Fetch everything known about the seed drug itself:

- Basic metadata: drug class (CNS DRUGS), route (oral), dosage form (tablet)
- Indications: FDA-approved + off-label conditions it treats (ICD-10 codes)
- Complexity: pill burden, multiple-daily-dosing, monitoring requirements

This is the baseline for comparison — a candidate is "better" only relative to what the seed already does.

### Step 2 — Evaluate the seed drug (~9s)

Score the seed on every axis that will also be used for candidates:

- **Safety:** contraindications against this patient's conditions, black box warnings, side effect count
- **Geriatric:** ACB score (anticholinergic burden), SLM score (sedative load), Beers Criteria flags
- **DDI:** interactions between the seed and each of the patient's current medications (one SQL query covering all drugs)

### Step 3 — Gather candidates — 3 sources, early-stop (~15–40s)

Three sources tried in order. If TXA or ETC alone returns ≥10 candidates the next source is skipped entirely.

| Source | What it finds | Breadth |
|---|---|---|
| **TXA** (Therapeutic Exchange Alternatives) | Drugs in the exact same therapeutic exchange group | Narrowest |
| **ETC** (Equivalent Therapeutic Class) | Drugs in the same class but not identical group | Broader |
| **Indication** | Drugs treating the same ICD-10 conditions (shared indications) | Widest |

For gabapentin (19 candidates): TXA + ETC fires, Indication never runs.

### Step 4 — Enrich all candidates (~20s, parallel)

For each candidate, fetch the same metadata fetched for the seed:

- `fetch_seed_medication_basic(gcn)` — class name, route, dosage form
- `_evaluate_medication_complexity(gcn)` — complexity info for the response payload
- `_fetch_indication_names_and_dxids(gcn)` — human-readable indication names

Originally sequential (45s). Now runs with a 6-worker `ThreadPoolExecutor`.

### Step 5 — Evaluate all candidates (~75s on local, ~12s on Cloud Run after Phase 2.5+2.6)

The dominant step. Runs in parallel across all candidates (6 workers). For each candidate:

1. **Safety eval** (SafetyProcessor): contraindications vs patient diagnoses, BBW, side effect burden. Retried up to 3× on DB fallback.
2. **Indication match** (IndicationProcessor): Jaccard similarity between candidate's ICD-10 codes and seed's ICD-10 codes. Candidates with zero shared indications are filtered out (`min_shared_indications: 1` in config).
3. **Geriatric eval** (GeriatricProcessor): ACB score + SLM score + Beers flags. SLM lookup is the complex piece — see the SLM 4-tier section below.
4. **DDI eval** (ClinicalDataProcessor): one SQL query for this candidate + all current meds (26 drugs total per query).
5. **Adjustments**: renal/hepatic dose adjustment flags, REMS program flags.

### Step 6 — Score, sort, return (~0s, pure in-memory)

Each candidate gets a composite score, penalties applied, then sorted and bucketed.

---

## Scoring and Ranking

### Composite score weights (currently)

| Dimension | Weight |
|---|---|
| Safety | 30% |
| Geriatric | 30% |
| Indication match | 25% |
| Source (TXA > ETC > Indication) | 15% |

Source boost: TXA +10pts, ETC +6pts, Indication +2pts.

### Penalty deductions (subtracted from composite)

| Trigger | Deduction |
|---|---|
| DDI contraindicated | -100 pts |
| DDI major | -50 pts |
| DDI moderate | -20 pts |
| DDI minor | -5 pts |
| Black box warning | -25 pts |
| Beers Criteria | -20 pts |
| ACB increase | -10 pts |
| Renal adjustment needed | -10 pts |
| Hepatic adjustment needed | -10 pts |
| REMS program | -12 pts |

### Hard gates (candidate blocked entirely if triggered)

| Gate | Config key |
|---|---|
| Contraindicated DDI with current medication | `gates.contraindicated_ddi: true` |
| Absolute contraindication against patient diagnosis | `gates.absolute_contra: true` |
| BBW for specific drug classes | `gates.bbw_for_classes: []` (empty = none blocked) |

### Recommendation bands

| Band | Score range |
|---|---|
| Recommended | ≥ 70 |
| Consider | 40–70 |
| Use With Caution | < 40 |

---

## SLM (Sedative Load Metric) — 4-Tier GCN Lookup

The GeriatricProcessor needs a sedative load score for each candidate drug. It tries 4 tiers in order:

### Tier 1 — JSON dict hit (fast, ~0ms)

Looks up `gcn_seqno` directly in `gcn_enriched_drugs.json` (in-memory dict, loaded at startup).

- If found: returns `sedative_activity_level` (None/Low/Moderate/High — score 0/1/2/3)
- GCN must match exactly. GCN 41805 (gabapentin 600mg) misses if only GCN 21414 (gabapentin 300mg) is in the dict.

### Tier 2 — Related-GCN DB query (~5–10ms when pool is warm)

Calls `_get_all_related_gcns(gcn)` on the FDB database via `slm_system.db`.

SQL joins `rgcnseq4_gcnseqno_mstr`, `rhiclsq2_hiclseqno_mstr`, `rrouted3_route_desc` grouped by `SAME_ROUTE_DIFF_STRENGTH` / `SAME_DRUG_DIFF_ROUTE`. Finds other GCNs for the same active ingredient (same `hicl_seqno`), then looks each up in the JSON dict. If any related GCN has a sedative score, returns it.

**Critical:** `slm_system.db` is the main `db_manager` pool (keepalives, always warm) — NOT `drug_matcher.db` (the separate small pool). If this were still using `drug_matcher.db`, cold-start or idle periods would silently fail this tier and fall to Tier 4, returning score=0.

### Tier 3 — Combination product check (DB query)

Checks if the drug is a combination product. Looks up active ingredients individually via `hicl_seqno`. If any component ingredient exists in the SLM JSON dict, returns the highest sedative score among components.

### Tier 4 — Name-based drug_matcher lookup (slow, currently disabled)

`drug_matcher.match(drug_name)` — tries to find the drug by name instead of GCN:

**A. DB candidate lookup:** Searches FDB by normalized drug name — returns ranked candidates with confidence scores.

**B. FAISS vector search** (only when `drug_matcher.vector_search is not None` AND `max_conf < 0.85`): Embeds the drug name using HuggingFace MedEmbed-large-v0.1 (1024-dim). Searches 33k pre-built embeddings for nearest neighbors. Adds vector candidates to the pool. CPU inference on Cloud Run: 12–46s per call.

**C. LLM disambiguation** (when `0.70 ≤ best_confidence ≤ 0.95` OR top-two candidates within 0.10 confidence): Sends the candidate pool to `fireworks-kimi-k2p6` to pick the best match. Model and `max_tokens` are configured in `drug_matcher_config.yaml` (`max_tokens: 8000` required — 500 would truncate JSON responses causing json-repair failures and wrong GCN picks).

After Phase 2.6, `drug_matcher.vector_search = None` at startup, so **Tier 4 returns score=0 immediately** for any drug that falls through Tiers 1–3. This is clinically accepted (no sedative risk assumed = not penalized). LLM disambiguation also never runs because the DB-only pool at sub-0.85 confidence without FAISS augmentation rarely meets the threshold.

---

## Drug Matcher (`FDBDrugMatcherWithLLM`)

Lives in `core_business_logic/src/drug_search/drug_matcher.py`.

Has its own FDB PostgreSQL connection pool (min=2, max=10, no keepalives) separate from the main `db_manager`. This pool is used for GCN-by-name DB lookups inside `match()`. `SLMLookupSystem.db` was **overridden** to use `db_manager` instead — only `match()` DB calls still use this smaller pool.

### `match()` flow

```
1. Cache hit? → return immediately (_match_cache dict, keyed by drug_name)
2. NDC exact match? → return (confidence 1.0)
3. DB name search → ranked candidates
4. If max_confidence < 0.85 AND vector_search is not None:
     FAISS embedding search → add vector candidates to pool
5. _select_best_match(candidates):
     if 0.70 ≤ best_conf ≤ 0.95 OR top2_gap < 0.10:
         LLM disambiguation
     else:
         return best candidate directly
6. Store in _match_cache, return
```

### FAISS lazy loading

`_init_vector_search()` in `drug_matcher.py` returns `None` immediately when `MEDALT_VECTOR_SEARCH_ENABLED` is not `"true"`. The import of `FAISSPharmaceuticalSearchEnhanced` is deferred to inside this method — so when FAISS is disabled, `torch`, `faiss`, and `transformers` are **never imported at all**, saving 60–120s of startup time and ~2GB RAM.

---

## Configuration Files

### `configs/med_alternative.yaml` — main config

Controls the ranker, all processors, scoring weights, and data paths.

```yaml
ranker:
  include_semantic: false           # FAISS disease search (separate from drug FAISS) — off
  max_etc_level: 40                 # ETC hierarchy depth limit
  same_route: true                  # ETC requires same route
  same_dosage_form: true            # ETC requires same dosage form
  min_shared_indications: 1         # Candidates with 0 shared indications filtered out
  weights:
    safety: 0.30
    geriatric: 0.30
    indication: 0.25
    source: 0.15
  bands:
    recommended: 70.0
    consider: 40.0
  gates:
    contraindicated_ddi: true
    absolute_contra: true
    bbw_for_classes: []
  penalties: { ... }                # See scoring section above
  boost_source: { TXA: 10.0, ETC: 6.0, Indication: 2.0 }
  early_stop:
    enabled: true
    min_results: 10
    quality_threshold: 70.0
    max_return: 50

processors:
  drug_matcher_config: configs/drug_matcher_config.yaml
  llm_config: configs/general_model_config.yaml
  acb_data_path: core_business_logic/data/acb/enriched_acb_dictionary.json
  slm_data_path: core_business_logic/data/slm/gcn_enriched_drugs.json
  slm_use_cot: false                # Has no runtime effect — geriatric_processor.py:403 hardcodes use_classifier=False
  TXAProcessor:
    require_same_route: false
    require_same_dosage_form: false
    allow_oral_solid_family: true
  ETCProcessor:
    max_level: 40
    require_same_route: true
    require_same_dosage_form: true
    allow_oral_solid_family: true
    min_classification_score: 3
  IndicationProcessor:
    min_shared_indications: 1
    use_semantic: false
    max_results: 50
    indication_min_jaccard: 0.0
    require_fda_approved: false
  GeriatricProcessor:
    use_beers: true / use_acb: true / use_slm: true
    fall_risk_threshold: 3
    beers_categories: [avoid, avoid_with_criteria, use_with_caution]
    acb_risk_levels: { low: 1, moderate: 2, high: 3 }
    slm_risk_levels: { low: 1.0, moderate: 2.0, high: 3.0 }
  SafetyProcessor:
    enable_bbw_gate: true
    include_side_effects: true
    concerning_effects_threshold: 5
    contraindication_severity_weights: { absolute: 100.0, relative: 50.0, precaution: 25.0 }
  ClinicalDataProcessor:
    check_current_meds: true
    include_food_interactions: false
    severity_levels: [contraindicated, major, moderate, minor]
    thresholds: { acb: 3, slm: 3.0, polypharmacy: 5 }
```

### `configs/drug_matcher_config.yaml` — drug disambiguation LLM

Used only by `FDBDrugMatcherWithLLM` (Tier 4 SLM lookup + any direct drug-name matching).

```yaml
llm:
  model: "fireworks-kimi-k2p6"      # Must match an LLM provider key supported by the submodule
  max_tokens: 8000                  # Must be >=8000 — disambiguation responses are large JSON
  temperature: 0.0
  use_llm_disambiguation: true
  llm_min_confidence: 0.70
  llm_max_confidence: 0.95
  llm_ambiguity_threshold: 0.10
database:
  host: <FDB_HOST> / user: <FDB_USER> / database: <FDB_DATABASE> / port: 5432
  min_connections: 2 / max_connections: 10
vector_search:
  index_path: "search_databases/bm25_expanded_pharma_search_index"
  model_name: "abhinand/MedEmbed-large-v0.1"
  use_gpu: false
```

!!! warning
    The real FDB host/user/database values are set via the `FDB_*` environment variables (see the table below) — they're intentionally redacted here rather than committed to a page that publishes publicly.

### `configs/general_model_config.yaml` — SLM Chain-of-Thought

Loaded by `LLMManager` for `SLMChainOfThoughtClassifier`. CoT is **hardcoded off** in `geriatric_processor.py:403` (`use_classifier=False`). Changes to `slm_use_cot` in the YAML have no runtime effect.

---

## Environment Variables — Complete Reference

All switches controlled from Cloud Run / deployment environment, no code change needed.

| Variable | Default | Effect |
|---|---|---|
| `MEDALT_EVAL_WORKERS` | `6` | Number of parallel workers in Step 5 `evaluate_all`. Set to 3 if FDB connection errors appear under load. |
| `MEDALT_FETCH_MEMO_TTL` | `3600` | TTL (seconds) for the fetcher DB-result cache. `0` = disabled (every call hits DB, original behavior). |
| `MEDALT_VECTOR_SEARCH_ENABLED` | `false` | `true` = FAISS enabled (slow, 12–46s/candidate, uses ~2GB RAM). Anything else = FAISS disabled at 3 levels: init skips loading, `drug_matcher.vector_search=None`, startup.sh skips pre-warm. |
| `JERIMED_EXTERNAL_SERVICE_URL` | unset | If set, router calls external GPU Cloud Run first and only falls back to in-process on failure. Unset to use the in-process path exclusively. |
| `MEDALT_WARMUP_GCN` | unset | If set, fires one background `orchestrate()` after init with this GCN as seed — warms the pool, FetchMemo, and code paths before the first real request. |
| `SEARCH_DB_GCS_PATH` | unset | GCS URI (e.g. `gs://my-bucket/search_databases`). If set, startup.sh downloads FAISS index files before the server starts. Only matters when `MEDALT_VECTOR_SEARCH_ENABLED=true`. |
| `FDB_HOST` | — | FDB PostgreSQL host (required) |
| `FDB_DATABASE` | — | FDB database name (required) |
| `FDB_USER` | — | FDB user (required) |
| `FDB_PASSWORD` | — | FDB password (required) |
| `FDB_PORT` | `5432` | FDB port |
| `FDB_SSLMODE` | — | PostgreSQL SSL mode |
| `PORT` | `8080` | Port uvicorn listens on |

### Rollback cheatsheet

| If you see | Fix |
|---|---|
| FDB connection errors under load | `MEDALT_EVAL_WORKERS=3` |
| Fetcher returning stale data | `MEDALT_FETCH_MEMO_TTL=0` |
| Need FAISS/SLM Tier 4 back | `MEDALT_VECTOR_SEARCH_ENABLED=true` + redeploy |
| Want to use the external jerimed service | Set `JERIMED_EXTERNAL_SERVICE_URL` |

---

## Connection Pool Architecture

Two separate PostgreSQL connection pools exist at runtime:

| Pool | Created by | Size | Keepalives | Used for |
|---|---|---|---|---|
| `db_manager` (`_TunedPostgreSQLManager`) | `medication_alternatives_service.py` | min=5, max=20 | Yes (idle=30s, interval=10s, count=3) | All fetcher DB calls, SLM Tier 2/3 (`slm_system.db`), all 6 processors |
| `drug_matcher.db` (`PostgreSQLManager`) | `FDBDrugMatcherWithLLM.__init__` | min=2, max=10 | No | Tier 4 name-based GCN lookups in `match()` only |

`slm_system.db` is explicitly overridden to `db_manager` after `SLMLookupSystem` construction — ensuring Tier 2 always has a warm connection even on cold start or after idle periods.

---

## Router Result Cache

`medication_alternatives_router.py` caches full endpoint responses in memory for **3600 seconds** (1 hour), keyed by `(gcn_seqno, patient_profile_hash)`. `config_overrides` requests bypass this cache. The fetcher TTL (`MEDALT_FETCH_MEMO_TTL`) is aligned to this — both expire at the same time, so a cache miss on the router also gets fresh DB data.

---

## Data Files (static, loaded at startup, read-only)

| File | Contents | Usage |
|---|---|---|
| `data/acb/enriched_acb_dictionary.json` | ACB scores keyed by GCN | Direct dict lookup in GeriatricProcessor |
| `data/slm/gcn_enriched_drugs.json` | Sedative activity levels keyed by GCN (trusted external source — never modify) | SLM Tier 1 dict lookup; also used by SLMLookupSystem Tier 3 |
| `data/slm/slm_gold_standards_cot.json` | CoT training examples | Loaded but unused at runtime (CoT hardcoded off) |
| `search_databases/bm25_expanded_pharma_search_index` | Pre-built FAISS index (33k drug embeddings) | Tier 4 vector search — only loaded when `MEDALT_VECTOR_SEARCH_ENABLED=true` |

---

## Performance History (local env, gabapentin 25 current meds, FDB dev DB)

All timings are local machine → FDB dev DB (Cloud Run is faster). Measurements are cold (no router/fetcher cache) unless noted.

| Milestone | Cold first request | Warm (router cache hit) | What changed |
|---|---|---|---|
| Baseline | 623.91s | — | — |
| Phase 1 (parallel workers 3–6, FetchMemo, pool sizing, YAML cache) | 533.56s | — | mom-backend only |
| Phase 2.1 (ComprehensiveAlternativeFinder singleton) | 477.58s | — | submodule |
| Phase 2.4 (parallel enrich+attach) | ~452s | — | mom-backend |
| Phase 2.5 (remove wasted `_find_alternatives()` from SLM eval — bug fix) | 146.66s | — | submodule |
| Phase 2.6 (FAISS disabled) + all fixes | **~113s** | **~6s** | mom-backend + submodule |
| Cloud Run (Phase 2.6, cold) | ~15s | — | different env — not directly comparable to local |

**Total local improvement: 623.91s → 113s (~82% faster cold, ~99% faster warm)**

Phase 2.5 was the dominant saving: `_get_slm_score()` was calling `lookup_by_gcn(gcn, include_alternatives=True)`, which triggered a full TXA search for each candidate's own alternatives — up to 86 alternatives × repeated DB queries per candidate — and then discarded all the results. One character fix (`True` → `False`).

The warm (router cache hit) time of ~6s reflects JSON serialization of the cached response, network overhead, and Python parsing — not any DB work.

---

## Test Curl — Gabapentin, Patient P-004 (25 current meds)

Local server on port 9561. Measured times: **cold ~113s**, **warm (router cache hit) ~6s**.

```bash
curl -s -X POST http://localhost:9561/webapi/api/v1/medication-alternatives \
  -H 'content-type: application/json' \
  -H 'X-Org-Database: longevity_db' \
  -H 'X-User-Role-ID: reviewer' \
  --data-raw '{
    "seed_drug": {"drug_name": "gabapentin", "gcn": "41805", "gcn_seqno": "41805"},
    "patient": {
      "patient_id": "P-004",
      "age": 61,
      "total_acb": 0,
      "total_slm": 0,
      "diagnoses": [
        "Schizoaffective disorder, bipolar type",
        "Pulmonary hypertension, unspecified",
        "Type 2 diabetes mellitus with both eyes affected by mild nonproliferative retinopathy without macular edema",
        "Extrapyramidal and movement disorder, unspecified",
        "Hypertensive heart and kidney disease with stage 3a chronic kidney disease",
        "Iron deficiency anemia secondary to inadequate dietary iron intake",
        "Slow transit constipation",
        "Type 2 diabetes mellitus with stage 3a chronic kidney disease",
        "Major depressive disorder, recurrent, moderate",
        "Atherosclerosis of native coronary artery of native heart with stable angina pectoris",
        "Other psychoactive substance dependence, uncomplicated",
        "Type 2 diabetes mellitus with hyperglycemia",
        "Overweight",
        "Restless leg syndrome",
        "Type 2 diabetes mellitus with hyperlipidemia",
        "Type 2 diabetes mellitus with diabetic peripheral angiopathy without gangrene",
        "Type 2 diabetes mellitus with diabetic polyneuropathy",
        "Seasonal allergies",
        "GERD without esophagitis"
      ],
      "allergies": ["Aspirin"],
      "current_medications": [
        {"gcn_seqno": "62867", "drug_name": "Lantus SoloStar Solution Pen-injector 100 UNIT/ML"},
        {"gcn_seqno": "62867", "drug_name": "Lantus SoloStar Solution Pen-injector 100 UNIT/ML"},
        {"gcn_seqno": "34731", "drug_name": "HumaLOG KwikPen Solution Pen-injector 100 UNIT/ML"},
        {"gcn_seqno": "34731", "drug_name": "HumaLOG KwikPen Solution Pen-injector 100 UNIT/ML"},
        {"gcn_seqno": "72489", "drug_name": "Jardiance Tablet 25 MG"},
        {"gcn_seqno": "34166", "drug_name": "ROPINIRole HCl Tablet 0.5 MG"},
        {"gcn_seqno": "19964", "drug_name": "Senna Tablet 8.6 MG"},
        {"gcn_seqno": "64302", "drug_name": "Vitamin D3 Tablet 2000 UNIT"},
        {"gcn_seqno": "4489",  "drug_name": "Acetaminophen Tablet 325 MG"},
        {"gcn_seqno": "4489",  "drug_name": "Acetaminophen Tablet 325 MG"},
        {"gcn_seqno": "47586", "drug_name": "Toprol XL Tablet Extended Release 24 Hour 25 MG"},
        {"gcn_seqno": "4589",  "drug_name": "Benztropine Mesylate Tablet 0.5 MG"},
        {"gcn_seqno": "38164", "drug_name": "Plavix Tablet 75 MG"},
        {"gcn_seqno": "46234", "drug_name": "Sertraline HCl Tablet 150 MG"},
        {"gcn_seqno": "1645",  "drug_name": "Ferrous Sulfate Tablet 325 (65 Fe) MG"},
        {"gcn_seqno": "41805", "drug_name": "Gabapentin Capsule 600 MG"},
        {"gcn_seqno": "29967", "drug_name": "Atorvastatin Calcium Tablet 10 MG"},
        {"gcn_seqno": "3758",  "drug_name": "LORazepam Tablet 1 MG"},
        {"gcn_seqno": "475",   "drug_name": "Nitroglycerin Tablet Sublingual 0.4 MG"},
        {"gcn_seqno": "13318", "drug_name": "MetFORMIN HCI Tablet 500 MG"},
        {"gcn_seqno": "39545", "drug_name": "Protonix Tablet Delayed Release 20 MG"},
        {"gcn_seqno": "29054", "drug_name": "Lactulose Solution 10 GM/15ML"},
        {"gcn_seqno": "66557", "drug_name": "CloZAPine Tablet 150 MG"},
        {"gcn_seqno": "41660", "drug_name": "Glucagon Emergency Kit 1 MG"},
        {"gcn_seqno": "1781",  "drug_name": "Glucose Gel 40"}
      ]
    },
    "options": {
      "include_text_report": true,
      "max_results": 100,
      "include_geriatric_analysis": true,
      "include_safety_analysis": true,
      "include_interaction_analysis": true,
      "include_cost_analysis": false,
      "severity_filter": null,
      "therapeutic_class_filter": null,
      "exclude_blocked": false
    }
  }' \
  | python3 -c "
import sys, json
data = json.load(sys.stdin)
print('success:', 'alternatives' in data)
print('alternatives_count:', len(data.get('alternatives', [])))
first = data.get('alternatives', [{}])[0]
print('first alt acb_score:', first.get('acb_score'))
print('seed acb:', data.get('seed_evaluation', {}).get('acb_score'))
"
```

---

## Known Hardcoded Behaviors (not configurable without code change)

- `SafetyProcessor` calls `fetch_rems_programs()` which does not exist on the fetcher — silently returns `[]`. Do not fix; fixing would change scoring behavior.
- SLM Chain-of-Thought classifier is always off regardless of the `slm_use_cot` YAML value (`geriatric_processor.py:403` hardcodes `use_classifier=False`).
- `create_custom_ranker()` in `medication_alternatives_service.py` calls `_apply_disable_vector_search(custom_ranker)` — this is redundant because the custom ranker shares the same `geriatric` object where `vector_search` is already `None`. No harm, just a no-op second call.
- HuggingFace model `abhinand/MedEmbed-large-v0.1` is baked into the Docker image at build time (`Dockerfile:47`) regardless of `MEDALT_VECTOR_SEARCH_ENABLED`. Model weights are on disk; they are not loaded into RAM when FAISS is disabled.
