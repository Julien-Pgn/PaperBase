# PaperBase audit — Sprint 1, Phase 0

Date: 2026-11-18. Commit audited: `7835d4c` (main, clean). Read-only audit.
DuckDB inspected through a byte-identical snapshot (the live file was locked by a running
`search_topic.py`, see Risk R1); snapshot deleted afterwards.

## 1. What actually exists

### Package and tooling
- Package is **already `paperbase`** (`src/paperbase/`, renamed in `5a5ec57` today). No `scirex`
  left in tracked code. Leftovers outside git: Docker image `scirex:dev`, network `scirex-net`.
  The project network used by `run_container.sh`/`run_vllm.sh` is `paperbase-net`.
- Tests: `pytest` in `paperbase:dev` (no GPU) -> **50 passed**; `ruff check` clean.
- `CONTRACT.md` sits at repo root, not `docs/`, and `.gitignore` has a bare `CONTRACT.md` entry,
  which also ignores `docs/CONTRACT.md` (gitignore patterns without `/` match at any depth).

### DuckDB `data/arxiv_metadata.duckdb` (12 GB, last write 2026-07-24)

| Table | Rows | Notes |
|---|---:|---|
| `papers` | 3,106,926 | `submitted_date`, `updated_date`, `version_count`; **no version string** |
| `authors` / `paper_authors` | 2.39M / 14.76M | `position` gives author order |
| `paper_categories` | 5,391,826 | `is_primary` |
| `ingestion_runs` | 3 | last Kaggle watermark 2026-07-17 |
| `abstract_embeddings` | 421,574 | BGE-M3 dense `FLOAT[1024]` + sparse `MAP(INTEGER, FLOAT)`, primary cat in cs.LG/CL/AI/CV only |
| `topic_subset` | 5,000 | quantization, run 2: 870 TRUE / 4,130 FALSE |
| `paper_local` | 3,935 | see below |
| `ocr_stats` | 35,300 | 2,130 papers; `error` = string `'false'` on every row; 128 pages hit the 8192-token cap |
| `benchmark_subset` / `genai_subset` | 1,000 / 7,000 | selection tables |
| `chunks` | 0 | stub, incompatible with the contract (see D-debts) |

Not present: `pipeline_runs`, `paper_stage_state`, `document_index_state`, any migrations ledger.
Sequences: `seq_run_id`, `seq_topic_run_id`.

### OCR corpus: DB flags vs disk

| Population | `.md` on disk | `paper_local.ocr_done` |
|---|---:|---|
| quantization (`keyword_for_ocr='quantization'`) | 863 | TRUE |
| benchmark_subset (labelled `keyword_for_ocr='genai_subset'`) | 996 | **FALSE** |
| other genai_subset / not in `paper_local` | 278 | FALSE / absent |
| **Total** | **2,137** | 863 TRUE |

All 2,137 exist in `papers`; only 1,541 have abstract embeddings (benchmark spans 13+
categories, e.g. cs.SE, cs.IR, cs.RO). Primary categories of the quantization set: cs.LG 514,
cs.CV 280, cs.CL 55, cs.AI 14. Version counts of quantization papers: 45% have >1 version.

### `data/processed/{arxiv_id}/`
`{id}.md`, `{id}.html` (raw Chandra layout HTML), `*_img.webp` figures. 146 MB of markdown total,
~68 KB/paper on average (est. ~17k tokens, ~20-30 chunks/paper, ~50k chunks for 2,137 papers).
Markdown shape that matters for chunking:
- `<!-- ===== Page N ===== -->` separators; paragraphs are cut across pages and around figures.
- Tables are **HTML `<table>`** (1,969/2,137 files), not pipe tables. Block math is `$$...$$`.
- Heading levels are inconsistent (`#### 4.2` under `## 4`), pseudo-headings exist
  (`### Algorithm 1 ...`), the title is repeated before supplementary material, `## References`
  is a section like any other. One file is 165 bytes (failed OCR).

## 2. Requirement -> existing module

| Sprint requirement | Existing | Status |
|---|---|---|
| `contract.py` | `CONTRACT.md` §2 | missing (copy) |
| `config.py` (pydantic-settings) | constants hard-coded per script (`DB_PATH`, `LOG_DIR`, `HEADERS`, sleep) | missing; `pydantic-settings` already a dep |
| DuckDB schema in code | DDL in `notebooks/db_build.ipynb` + inline in `embed_abstracts.py`, `search_topic.py` | missing (move to versioned SQL) |
| `get_paper` | `papers` + `paper_authors` + `paper_categories` | data reusable, query missing |
| `search_papers` | `hybrid_search.py` (`dense_scores`, `sparse_scores`, `rrf_fuse`), `embed.load_corpus_slice`, `embed.load_model` | reusable as-is; needs date/category filters + long-lived in-memory corpus |
| BGE-M3 encoding | `embed.py` (`load_model`, `embed_texts`) | reusable as-is |
| FastAPI service | none; `fastapi` **not installed** in image (uvicorn, qdrant-client are, via `rag` extra) | missing; add dep + rebuild image |
| Golden set v1 (JSONL) | `quantization_golden_set_v2.parquet` (paper-level relevance, not QA) | missing (different task) |
| Chunker | none | missing |
| `Document` model | none | missing |
| `document_index_state` | `paper_local` (+ disk) | missing; backfill from both |
| `chunks` table | empty stub, `chunk_id BIGINT`, `UNIQUE(arxiv_id, chunk_index)`, `section VARCHAR` | adapt: drop and recreate (empty) |
| Qdrant service | `run_vllm.sh` pattern, `paperbase-net` | missing; `qdrant-client` 1.18 installed |
| `index_papers.py` | `embed_abstracts.py` shape (argparse, dual logging, banner, bulk insert) | pattern reusable |
| arXiv HTML fetch | `fetch_files.py` (`HEADERS`, `HTTP_RETRY`, 3 s sleep) | adapt (retry/UA reusable, 4xx retried needlessly) |
| HTML parser | `bs4` + `lxml` installed | missing |
| PyMuPDF text fallback | `pymupdf` installed; `pipeline.pdf_to_images` uses it | missing |
| Batch OCR (`--ocr`) | `ocr/modern/pipeline.py`, `scripts/run_ocr.py` | reusable; selection is by `keyword_for_ocr`, needs a `needs_ocr` queue selector |
| `retrieve_chunks` | `rrf_fuse` | adapt: it fuses full-length score arrays, Qdrant returns top-N lists |
| `eval_retrieval.py` | AP/precision logic in `notebooks/topic_retrieval.ipynb` | missing |

## 3. Debts
1. Schema exists only in a notebook + inline DDL; no migration ledger.
2. `paper_local.ocr_done` false for 1,274 papers that are OCR'd on disk; benchmark papers carry
   `keyword_for_ocr='genai_subset'`. `run_ocr.py` skips on `.md` existence, so reruns never repair flags.
3. `ocr_stats.error` stores `str(False)`; truncated pages (128) are not surfaced per paper.
4. ROADMAP says Phase 2 (pipeline state tables) "done"; PROJECT says it was skipped, and the DB
   confirms it does not exist.
5. `chunks` stub violates contract rule 1 (string id, multiple chunker versions).
6. No arXiv version recorded for any fetched PDF (`fetch_pdf` uses unversioned URLs).
7. `ocr/modern/pipeline.py` imports `chandra` at module top (not lazy); the API must not import it.
8. `extract_latex.py` + 916 extracted LaTeX trees have no downstream consumer.
9. `run_container.sh streamlit` points to `src/paperbase/demo/app.py`, which does not exist;
   empty packages `extraction/`, `llm/`, `rag/`, `utils/`.
10. `coverage.xml` is tracked in git; `data/ocr_hf`, `data/ocr_vllm` still on disk (PROJECT says removed).
11. PROJECT.md says benchmark = 900 papers; table has 1,000 (996 OCR'd).
12. `recommend_top_k` outlier fragility (already documented in PROJECT.md).

## 4. Risks
- **R1 Single-writer lock.** DuckDB allows one read-write process, and a read-only open from
  another process also fails while it is held. Observed during this audit: `search_topic.py`
  (started 22:06, ETA ~01:00) holds the lock, and on completion it rewrites `topic_subset`
  quantization rows. Consequence: the API server must be the only DB owner while it runs;
  batch scripts run only when the API is stopped (or go through the API).
- **R2 Two execution modes.** Online API (BGE-M3 query encoding, ~1.1 GB VRAM fp16, or CPU)
  vs offline batch (vLLM Chandra takes `--gpu-memory-utilization 0.8` of 16 GB; chunk embedding
  takes BGE-M3 at batch size). Never concurrent. Needs an explicit guard, not just discipline.
- **R3 API memory/startup.** Abstract search loads 421k dense vectors (~1.7 GB float32) + 421k
  sparse dicts into RAM: ~90 s load measured in `search_topic.log`. Sparse scoring is a Python
  loop over 421k docs per query.
- **R4 Dependencies.** `fastapi` absent from the image -> pyproject change + `./run_container.sh build`.
  Must not redeclare container packages (pydantic, httpx, starlette is pulled by fastapi: check
  it does not downgrade anything). Qdrant server image version must match client 1.18.
- **R5 Versions.** No version for existing OCR text; contract `chunk_id` embeds the version.
- **R6 Stale environment.** Running job uses image `scirex:dev` on `scirex-net` (pre-rename);
  the new Qdrant container must join `paperbase-net`.
- **R7 OCR quality.** Page-split paragraphs, HTML tables, 128 truncated pages, one near-empty
  file: golden quotes may not match OCR text verbatim (whitespace, escaping like `\_`, `\*`).

## 5. Decisions requested (recommendation first)

**D1 Rename `scirex` -> `paperbase`.** Already done in code. Remaining: replace `<package>` by
`paperbase` in the contract paths, rebuild image, retire `scirex:dev` / `scirex-net` after the
running job ends. Use `paperbase-net`, not `scirex-net` (prompt wording is outdated).

**D2 Schema location.** Recommend `src/paperbase/db/migrations/NNN_name.sql` + a small
`apply_migrations(conn)` that records applied files in a `schema_migrations` table. `001_baseline.sql`
= current real schema (from the DB, not the notebook) with `IF NOT EXISTS`. Alternatives: a single
idempotent `schema.sql` (simpler, but cannot express "drop the old chunks stub" or column
renames safely); Alembic (overkill, weak DuckDB support).

**D3 Qdrant.** Recommend `run_qdrant.sh` (same shape as `run_vllm.sh`), pinned `qdrant/qdrant`
image matching client 1.18, on `paperbase-net`, port 6333 bound to 127.0.0.1, storage in
`data/qdrant/`. Tests use `QdrantClient(":memory:")` (local mode, no network).

**D4 Fusion.** Recommend client-side RRF: two Qdrant queries (dense top-N, sparse top-N), fused by a
rank-list variant of `rrf_fuse` with configurable `k`. Same formula as abstract search, and dense /
sparse / hybrid are evaluated from the same two calls. Alternative: Qdrant native `prefetch` +
RRF fusion (fewer lines, one round trip, less control/visibility).

**D5 Corpus scope for Sprint 1.** Recommend indexing all 2,137 OCR'd papers (quantization + benchmark
+ genai leftovers), with `document_index_state` backfilled from disk + `paper_local`, without
mutating `paper_local`. Alternative: quantization only (863).

**D6 Version of existing OCR text.** Recommend `version=None` for existing OCR (honest: unknown),
explicit version for HTML-acquired papers. Needs a contract clarification of the `chunk_id` format
when version is None (proposed: `2306.00978:c1:0007`). Alternatives: assume `v{version_count}`
(wrong for papers revised after fetch); query arXiv for every paper (~2 h of rate-limited calls).

**D7 `search_papers` backend.** Recommend Phase 1 = existing in-RAM numpy search loaded once at API
startup (reuses code as the plan asks), measure latency; revisit in Phase 4 by putting abstracts
in a second Qdrant collection if it is too slow. Query encoder device configurable (`cuda`/`cpu`).

**D8 Contract file.** Move `CONTRACT.md` to `docs/CONTRACT.md` and remove the `.gitignore` entry
(otherwise the contract is never versioned).
