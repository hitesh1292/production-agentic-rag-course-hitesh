# Week 1 Reading Notes

File-by-file analysis from the initial codebase read.

---

## README.md
**What it does:** Course overview describing a 7-week progressive RAG system build from Docker infrastructure through LangGraph agents.
**Assumes:** The reader has Docker Desktop, Python 3.12+, UV, 8GB RAM, and 20GB disk.
**Breaks if wrong:** Students develop the wrong mental model of the architecture and don't know which week to start on or what each service does.

---

## compose.yml
**What it does:** Defines and wires together 15 Docker containers (api, postgres, opensearch, opensearch-dashboards, airflow, ollama, redis, clickhouse, langfuse-web, langfuse-worker, langfuse-postgres, langfuse-redis, langfuse-minio) on a shared `rag-network` bridge.
**Assumes:** `.env` file exists with `LANGFUSE_SALT`, `LANGFUSE_ENCRYPTION_KEY`, `LANGFUSE_NEXTAUTH_SECRET`, `LANGFUSE_MINIO_ACCESS_KEY/SECRET_KEY`, `LANGFUSE_REDIS_PASSWORD`; Docker daemon is running; ports 8000, 5432, 9200, 5601, 8080, 11434, 6379, 3001, 3030, 5433, 6380, 9090, 8123 are free.
**Breaks if wrong:** Services fail to start (missing env vars), fail to talk to each other (wrong hostnames — note container-internal URLs like `http://opensearch:9200` differ from localhost URLs), or data is lost on restart (missing volume definitions).

---

## Makefile
**What it does:** Provides shorthand `make` targets wrapping `docker compose` and `uv` commands for common dev tasks (start, stop, health, test, lint, format, clean).
**Assumes:** `docker`, `docker compose`, `uv`, `jq`, and `curl` are installed and on `$PATH`.
**Breaks if wrong:** Make targets silently run wrong commands; health checks check the wrong port (currently checks `/health` but API exposes `/api/v1/health`).

---

## pyproject.toml
**What it does:** Declares the Python 3.12 project, all production and dev dependencies, and tool config for ruff (linting), pytest (async mode, `.env.test`), and mypy (errors ignored).
**Assumes:** `uv` is installed; Python 3.12 is available.
**Breaks if wrong:** Missing dependencies cause `ImportError` at runtime; wrong tool config means lint/type errors go undetected or tests load the wrong env file.
**Notable:** `requests` is listed as a production dependency despite CLAUDE.md forbidding it — it is likely pulled in transitively and was not explicitly added.

---

## Dockerfile
**What it does:** Multi-stage build — stage 1 uses the `uv` image to install production-only dependencies; stage 2 copies the venv into `python:3.12-slim` and runs uvicorn with 4 workers.
**Assumes:** `src/` directory exists at build time; `uv.lock` is committed and consistent with `pyproject.toml`.
**Breaks if wrong:** `uv sync --frozen` fails if `uv.lock` is out of sync; app fails to import if `src/` is missing; 4 workers may over-commit memory on low-RAM machines.

---

## src/config.py
**What it does:** Defines all application configuration as nested Pydantic `BaseSettings` classes, read from `.env` with `__` as the nested delimiter (e.g. `REDIS__HOST` → `settings.redis.host`); `get_settings()` returns a fresh instance each call.
**Assumes:** `.env` file or environment variables exist; `POSTGRES_DATABASE_URL` starts with `postgresql://` or `postgresql+psycopg2://`; the PDF cache directory path is writable (it is created on validation).
**Breaks if wrong:** Pydantic raises `ValidationError` on startup; wrong URL format raises a `ValueError` before any service starts; misconfigured settings silently use defaults (e.g., pointing at `localhost` when inside Docker requires the container hostname).

---

## src/exceptions.py
**What it does:** Defines the entire exception hierarchy for the app — repository, parsing, PDF, OpenSearch, arXiv, LLM, and pipeline errors — as plain Python classes with no logic.
**Assumes:** Nothing at runtime (pure definitions).
**Breaks if wrong:** Catch blocks in services match the wrong exception type and errors propagate silently; week-tagged comments indicate some exceptions are stubs and will be filled later.

---

## src/database.py
**What it does:** Module-level singleton that lazily creates and caches a `BaseDatabase` instance via `get_database()`; exposes `get_db_session()` as a context manager — used by Airflow DAGs and scripts, not by FastAPI (which uses `dependencies.py` instead).
**Assumes:** `make_database()` succeeds (PostgreSQL running, env vars set).
**Breaks if wrong:** Airflow DAGs or standalone scripts can't access the database; the module-level `_database` global violates the "no global variables" rule in CLAUDE.md — be aware this is intentional legacy design for non-FastAPI contexts.

---

## src/dependencies.py
**What it does:** FastAPI dependency injection hub — pulls every service from `request.app.state` (set during lifespan) and exports typed `Annotated` aliases (`OpenSearchDep`, `OllamaDep`, etc.) for use in router function signatures; `get_settings()` is cached via `@lru_cache`; `get_agentic_rag_service` is constructed per-request from injected dependencies.
**Assumes:** `app.state` was fully populated during lifespan in `main.py`; `AgenticRAGService.factory` exists at `src/services/agents/factory.py`.
**Breaks if wrong:** Every router that uses a `*Dep` alias raises `AttributeError` at request time; if `make_agentic_rag_service` is expensive, per-request construction is a performance issue.

---

## src/middlewares.py
**What it does:** Contains two standalone logging helper functions (`log_request`, `log_error`) that are not wired into FastAPI's middleware system — the file is a placeholder for future BaseHTTPMiddleware integration.
**Assumes:** Nothing. These functions are never called automatically.
**Breaks if wrong:** Nothing currently breaks because these functions are not registered as middleware; adding them incorrectly as middleware would affect every request.

---

## src/schemas/api/health.py
**What it does:** Defines `HealthResponse` and `ServiceStatus` Pydantic models returned by the health check endpoint.
**Assumes:** Nothing (pure data types).
**Breaks if wrong:** Health check endpoint fails serialization; monitoring systems parsing the JSON response break.

## src/schemas/api/search.py
**What it does:** Defines `SearchRequest`, `HybridSearchRequest`, `SearchHit`, and `SearchResponse` — the full request/response contract for search endpoints.
**Assumes:** Nothing (pure data types).
**Breaks if wrong:** Routers reject valid requests or serialize responses incorrectly; the `from_` / `from` alias pattern is load-bearing — both names must work.

## src/schemas/api/ask.py
**What it does:** Defines `AskRequest`, `AskResponse`, `AgenticAskResponse` (extends `AskResponse` with reasoning steps), `FeedbackRequest`, and `FeedbackResponse`.
**Assumes:** Nothing (pure data types).
**Breaks if wrong:** RAG endpoints reject valid queries or don't return expected fields; `AgenticAskResponse` inheriting `AskResponse` means any field change in the base breaks both endpoints.

## src/schemas/arxiv/paper.py
**What it does:** Defines the paper data lifecycle: `ArxivPaper` (raw API response), `PaperBase` (shared fields), `PaperCreate` (includes optional PDF content), `PaperResponse` (API output with UUID and timestamps), `PaperSearchResponse` (paginated list).
**Assumes:** Nothing (pure data types).
**Breaks if wrong:** Mismatches between `PaperCreate` and the `Paper` ORM model cause `model_dump()` → ORM insert failures.

## src/schemas/ollama.py
**What it does:** Defines `RAGResponse` — the structured Pydantic model the Ollama client parses LLM JSON output into (answer, sources, confidence, citations).
**Assumes:** The LLM returns valid JSON matching this shape (stripped of markdown fences first per CLAUDE.md rules).
**Breaks if wrong:** JSON parse fails silently or returns wrong fields to callers.

## src/schemas/embeddings/jina.py
**What it does:** Defines the request/response shapes for the Jina AI embeddings API (`jina-embeddings-v3`, 1024-dim, float).
**Assumes:** Jina API returns this exact structure.
**Breaks if wrong:** Embedding requests fail validation; vector dimensions don't match the 1024-dim OpenSearch index mapping.

## src/schemas/pdf_parser/models.py
**What it does:** Defines the structured output of PDF parsing: `PaperSection`, `PaperFigure`, `PaperTable`, `PdfContent`, `ArxivMetadata`, and `ParsedPaper` (the combined result).
**Assumes:** Docling parses documents into sections, figures, tables.
**Breaks if wrong:** PDF parser output doesn't match, causing chunking failures downstream.

## src/schemas/indexing/models.py
**What it does:** Defines `ChunkMetadata` (position, word counts, overlap, section title) and `TextChunk` (text + metadata + arxiv_id + paper_id) — the unit of data indexed into OpenSearch.
**Assumes:** Nothing (pure data types).
**Breaks if wrong:** OpenSearch documents are missing fields the search query builder expects (e.g., `chunk_id`, `section_title`).

## src/schemas/database/config.py
**What it does:** `PostgreSQLSettings` — a second, smaller settings class for DB configuration; used specifically by `db/factory.py` to construct `PostgreSQLDatabase`.
**Assumes:** `POSTGRES_*` env vars or defaults.
**Breaks if wrong:** DB factory can't construct a valid connection config.

---

## src/models/paper.py
**What it does:** Defines the single `Paper` SQLAlchemy ORM model (table: `papers`) with UUID primary key, arXiv metadata columns, JSON columns for authors/categories/sections/references, and PDF processing status fields.
**Assumes:** `Base` from `db/interfaces/postgresql.py` exists; PostgreSQL is running and the DB user has CREATE TABLE permission.
**Breaks if wrong:** Table doesn't exist and all DB operations fail; changing column types after first run requires a migration (no Alembic migrations are present — `create_all` is used instead, so schema changes to existing tables silently don't apply).

---

## src/db/interfaces/base.py
**What it does:** Defines two ABCs: `BaseDatabase` (startup/teardown/get_session) and `BaseRepository` (create/get_by_id/update/delete/list) — the contract all DB implementations must fulfil.
**Assumes:** Nothing (abstract definitions only).
**Breaks if wrong:** Implementations can omit required methods without Python raising an error until the missing method is called; `PaperRepository` does NOT inherit from `BaseRepository` — this contract is partially unenforced.

## src/db/interfaces/postgresql.py
**What it does:** `PostgreSQLDatabase` — concrete implementation that creates a SQLAlchemy engine with connection pooling (pool_size=20, pool_pre_ping=True), creates all tables on startup via `Base.metadata.create_all`, and yields sessions with auto-rollback on exception.
**Assumes:** PostgreSQL is reachable at the configured URL; `Base` is imported by all model files before `startup()` is called (so `create_all` sees all models).
**Breaks if wrong:** Models imported after `startup()` don't get their tables created; connection pool exhaustion if sessions aren't properly closed; schema changes to existing tables are silently ignored by `create_all`.

## src/db/factory.py
**What it does:** `make_database()` — reads settings, constructs `PostgreSQLSettings`, instantiates and starts `PostgreSQLDatabase`; called once in `main.py` lifespan and once lazily in `database.py`.
**Assumes:** `get_settings()` succeeds; PostgreSQL container is healthy.
**Breaks if wrong:** If called multiple times, creates multiple engine instances and connection pools — currently safe because `main.py` calls it once and `database.py` uses a singleton guard.

---

## src/repositories/paper.py
**What it does:** `PaperRepository` — all database queries for papers: create, get by arxiv_id, get by UUID, list (sorted by date desc), count, filter by processing status, upsert (check-then-create-or-update), and processing statistics.
**Assumes:** A live `sqlalchemy.orm.Session` is passed in; the `Paper` table exists.
**Breaks if wrong:** `upsert()` has a race condition — two concurrent calls with the same arxiv_id can both pass the `get_by_arxiv_id` check and both try to `create()`, causing a unique constraint violation; `PaperRepository` doesn't inherit `BaseRepository` so it is not enforced by the ABC.

---

## src/routers/ping.py
**What it does:** `GET /api/v1/health` — checks database connectivity (SELECT 1), OpenSearch health and document count, and Ollama availability; returns `HealthResponse` with "ok" or "degraded" status and per-service details.
**Assumes:** `DatabaseDep`, `OpenSearchDep`, `SettingsDep` are correctly populated in `app.state`.
**Breaks if wrong:** Load balancers or monitoring receive incorrect health signals; notably, Ollama is constructed fresh per-request here (not from `app.state`) — inconsistent with all other services.

## src/routers/hybrid_search.py
**What it does:** `POST /api/v1/hybrid-search/` — generates an optional query embedding (graceful fallback to BM25 if Jina fails), calls `opensearch_client.search_unified()`, and returns a `SearchResponse` with chunk-level hits.
**Assumes:** OpenSearch is healthy; `EmbeddingsDep` and `OpenSearchDep` are populated.
**Breaks if wrong:** Returns 503 if OpenSearch is down; returns 500 on all other errors; if embedding fails it silently falls back to BM25 without surfacing that information clearly.

## src/routers/ask.py
**What it does:** Two routers — `POST /api/v1/ask` (sync) and `POST /api/v1/stream` (Server-Sent Events); both: check Redis cache first, retrieve chunks via hybrid search, build RAG prompt, call Ollama, store to cache, return answer with sources; stream sends metadata packet first then token-by-token chunks then a done signal.
**Assumes:** `OpenSearchDep`, `EmbeddingsDep`, `OllamaDep`, `LangfuseDep`, `CacheDep` are populated; Ollama has the requested model loaded.
**Breaks if wrong:** Cache miss path hits Ollama which may be slow or unresponsive; if `chunks` is empty it returns a canned "no information found" response rather than raising an error; streaming errors are sent as a `data: {"error": ...}` SSE event, not an HTTP error code.

## src/routers/agentic_ask.py
**What it does:** `POST /api/v1/ask-agentic` — delegates entirely to `AgenticRAGService.ask()`; `POST /api/v1/feedback` — submits a score+comment to Langfuse against a `trace_id` returned from a prior response.
**Assumes:** `AgenticRAGDep` is correctly built per-request via `make_agentic_rag_service`; Langfuse is running and has the trace_id.
**Breaks if wrong:** `AgenticRAGDep` is constructed per-request from injected services — if any injected service is None the construction silently fails at call time; feedback without a valid trace_id will fail at Langfuse level.

---

## src/gradio_app.py
**What it does:** A standalone Gradio web UI (port 7861) that streams answers from the FastAPI `/api/v1/stream` endpoint via httpx; renders answer tokens in real-time with source links.
**Assumes:** FastAPI server is running at `http://localhost:8000`; `AVAILABLE_CATEGORIES` is hardcoded to `["cs.AI", "cs.LG"]`.
**Breaks if wrong:** Hardcoded `API_BASE_URL` means it can't run against a remote server without code change; `json.JSONDecodeError` lines are silently skipped — malformed SSE data is invisible.

---

## src/main.py
**What it does:** The FastAPI app entry point — defines the `lifespan` context manager that initialises all 9 services into `app.state` in dependency order, registers 5 routers, and starts/stops the Telegram bot; then creates the `FastAPI` app with this lifespan.
**Assumes:** All factory functions succeed (databases, OpenSearch, etc. are reachable); `.env` is loaded.
**Breaks if wrong:** Any factory failure aborts startup entirely; the order of `app.state` assignments matters — Telegram init uses `opensearch_client`, `embeddings_service`, `ollama_client`, `cache_client`, and `langfuse_tracer` which must be set before the Telegram factory is called; if `agentic_ask.router` is included without its prefix, it doesn't get `/api/v1` — confirmed: it sets its own `prefix="/api/v1"` in the router definition.

---

## notebooks/week1/README.md
**What it does:** Orientation guide for the Week 1 Jupyter notebook — explains what each section covers (env checks, service setup, Ollama testing), time estimates, and links to the blog post.
**Assumes:** Reader will run the notebook from the project root after `docker compose up`.
**Breaks if wrong:** Students attempt the notebook before Docker is up, getting confusing connection errors; Ollama models are described as optional for Week 1 which is correct.

---

## week1_setup.ipynb (root-level copy)
**What it does:** 39-cell Jupyter notebook walking through prerequisite checks (Python version, Docker, Docker Compose, UV), then step-by-step service health verification for each container (PostgreSQL, OpenSearch, Airflow, Ollama, FastAPI).
**Assumes:** Notebook is run with the project root as the working directory; Docker services are already up (`docker compose up -d`); Python 3.12+ and UV are installed on the host.
**Breaks if wrong:** Cells that call `subprocess.run(["docker", ...])` fail if Docker isn't on PATH; the notebook auto-detects whether it's run from the notebooks/week1/ directory or the project root — if run from elsewhere, path resolution breaks.

---

## src/services/ subdirectories (not yet read)

The following service subdirectories exist inside `src/services/`. Contents will be read in future weeks:

- `agents/` — LangGraph agentic RAG workflow (Week 7)
- `arxiv/` — arXiv API client with rate limiting (Week 2)
- `cache/` — Redis cache client (Week 6)
- `embeddings/` — Jina AI embeddings client (Week 4)
- `indexing/` — Text chunker + hybrid indexer (Week 4)
- `langfuse/` — RAG pipeline tracing (Week 6)
- `ollama/` — Local LLM client + prompt builder (Week 5)
- `opensearch/` — Search client, query builder, index config (Week 3)
- `pdf_parser/` — Docling PDF parser (Week 2)
- `telegram/` — Telegram bot integration (Week 7)
- `metadata_fetcher.py` — Ingestion orchestrator (Week 2)
