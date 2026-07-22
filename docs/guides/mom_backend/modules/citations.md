# Citations

The Citations module resolves citation references embedded in clinical intelligence reports (AHFS drug monographs, FDA prescribing-information sections, and clinical guideline excerpts) and renders them as styled, human-readable HTML for display in a citation viewer modal. Guideline citations are stored as vector-embedded chunks in Weaviate and are looked up by object UUID rather than by semantic search — the vector index is queried with exact-match ID filters, and each guideline object also carries a source PDF/page reference so the module can serve the original PDF page or a rendered PNG alongside the extracted text.

## Endpoints

| Method | Path | Purpose |
|---|---|---|
| GET | `/webapi/api/v1/citations/view` | Look up a citation by `citation_id` within a patient's stored intelligence report (`patient_id` + `run_id`) and return rendered HTML (AHFS, FDA PI, or guideline content). Always returns HTTP 200 with an HTML error card on failure, to avoid breaking the frontend's citation modal. |
| GET | `/webapi/api/v1/citations/view_ref` | Look up and render a citation directly by reference fields (`source=ahfs\|fda_label`, plus `ahfs_mono`/`ahfs_sect` or `fda_drug_id`/`loinc`/`section_title`), without needing a stored intelligence report. Returns JSON error bodies on failure. |
| GET | `/webapi/api/v1/citations/guideline_pdf/{uuid}` | Serve the source guideline PDF for a Weaviate guideline chunk `uuid` — either a single extracted page (`mode=page`, default) or the full document (`mode=full`) |
| GET | `/webapi/api/v1/citations/guideline_image/{uuid}` | Serve the guideline chunk's source page as a rendered PNG image |

## Data model

- `DrugSection` (`drug_sections` table) — a relational row representing one section of an FDA drug label: `drug_id`, `loinc_code` (LOINC section code), `section_title`, `section_text`/`section_html`, `section_order`. Used to look up FDA prescribing-information and FDA-label sections by drug, LOINC code, or section title/number pattern.
- `AHFSSection` (in-memory dataclass, not a DB table) — an AHFS monograph section assembled at query time from FDB reference tables, holding `ahfs_mono`, `ahfs_sect`, hierarchy `level`/`hierarchical_path`, `section_title`, a computed `clinical_priority`/`priority_description`, concatenated `section_text`, and text segment/length counts.
- Weaviate `GuidelineChunk` collection (vector index) — one object per guideline text chunk, addressed by UUID, with properties: `text`/`content`, `content_type`, `page_number`, `guideline_id`, `guideline_source`, `specialty`, `recommendation_class`, `evidence_level`, `medication_names`, `medical_conditions`, `confidence`. Objects are fetched by exact ID filter, not by similarity search, in all code paths observed.

## Notes

- AHFS (American Hospital Formulary Service) monographs are sourced from First Databank (FDB) reference tables; the manager auto-detects whether those tables exist in lowercase or uppercase naming and disables AHFS citations gracefully if none are found.
- A hardcoded `clinical_priority`/`priority_description` mapping (e.g. "Cautions" → priority 1 "CRITICAL", "Drug Interactions" → priority 2, geriatric/renal/hepatic subsections boosted) is computed via SQL `CASE` expressions at query time, not stored.
- `CitationAHFSManager.find_section_by_title` is a fallback used when an AHFS monograph section-number reference is stale (AHFS releases renumber sections between FDB versions) — it re-resolves the correct section by matching the stored title text instead.
- Weaviate connection config requires both `WEAVIATE_URL` and `WEAVIATE_API_KEY` env vars; if either is missing, the manager logs a warning and leaves the client unset rather than raising, so guideline citations silently become unavailable rather than failing hard. A related embedding config class defines a MedCPT-based embedding model, though the Weaviate manager itself performs no embedding calls — it only does ID-based fetches.
- `CitationService.view_fda_pi_content` works around a `ZeroDivisionError` in the shared `core_business_logic` FDA dedup logic (triggered when two same-numbered sections both have whitespace-only text within the first 1000 chars) by falling back to a local safe implementation with an added whitespace filter.
- `/view` intentionally always responds with HTTP 200 (embedding the real status in the HTML body) because the frontend's citation modal uses axios and a non-200 response would throw before the UI could render an error state; `/view_ref` and the PDF/image endpoints instead return standard JSON/HTTP error codes (400/404/500/503).
- PDF/image serving depends on a local guideline-PDF directory of source PDFs matched to a Weaviate `guideline_source`/`specialty` by filename normalization and word-overlap scoring; image conversion tries `pdf2image` (needs poppler) first and falls back to PyMuPDF (`fitz`).
