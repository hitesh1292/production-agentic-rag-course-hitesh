# DECISIONS.md

Architectural decisions identified from reading the codebase.
Updated as the project progresses.

---

## 1. Layered Architecture: Routers → Services → Repositories → DB

**The layers:**
```
HTTP Request
    ↓
routers/        — HTTP concerns only (parse request, return response, raise HTTPException)
    ↓
services/       — Business logic (search, LLM, embeddings, caching, tracing)
    ↓
repositories/   — Data access patterns (queries, upserts, filters)
    ↓
db/             — Connection management, sessions, schema creation
    ↓
models/         — ORM table definitions
```

**Why each layer exists separately:**

- **Routers** exist to isolate HTTP concerns from business logic. They speak FastAPI (request/response models, HTTPException, dependency injection) and nothing else. If you swap from FastAPI to another framework, only routers change.

- **Services** contain the real work — hybrid search, LLM calls, PDF parsing, caching, tracing. They are framework-agnostic Python classes that can be tested without an HTTP server. They are initialized once in `main.py` lifespan and injected via `dependencies.py`, so they carry no per-request state.

- **Repositories** isolate all SQL from services. Services pass a `Session` to a repository and call named methods (`upsert`, `get_by_arxiv_id`). Services never write raw SQL or SQLAlchemy queries. This means if the DB schema changes, only the repository changes.

- **DB layer** (`db/interfaces/`, `db/factory.py`) isolates connection management, pooling, and table creation. `BaseDatabase` is an ABC so a future test implementation could swap in SQLite without changing services or repositories.

- **Models** (`models/paper.py`) are pure ORM definitions — no business logic. They import `Base` from `postgresql.py` so that `Base.metadata.create_all` can discover all tables.

**The practical consequence:** A router never touches a SQLAlchemy model directly. It calls a service. The service opens a session via `database.get_session()`, instantiates a repository with that session, and calls repository methods. Results are mapped to Pydantic schemas before being returned to the router.

---

## 2. All Services Stored in `app.state`, Injected via `dependencies.py`

**Decision:** Services are created once in the FastAPI lifespan function and stored in `app.state`. Routers access them via typed `Annotated` dependency functions in `dependencies.py`.

**Why:** FastAPI dependency injection with `request.app.state` means:
1. Services are singletons — connection pools, API clients, and LLM clients are expensive to create per-request.
2. The dependency graph is visible and testable — tests can replace `app.state.opensearch_client` with a mock.
3. Routers declare exactly what they need via their signature, which is self-documenting.

**The exception:** `AgenticRAGService` is constructed per-request in `get_agentic_rag_service` because it takes other injected services as arguments. This is a composition pattern, not a performance concern — the underlying services it wraps are still singletons.

---

## 3. Single Hybrid OpenSearch Index (Not Separate BM25 + Vector Indices)

**Decision:** One index (`arxiv-papers-chunks`) supports BM25, vector-only, and hybrid (RRF) search simultaneously. The search mode is chosen at query time, not at index time.

**Why:**
- Simpler operations — one index to manage, back up, and monitor.
- Reciprocal Rank Fusion (RRF) is applied at query time by OpenSearch's search pipeline, so no data duplication.
- Allows graceful degradation: if Jina embeddings are unavailable, the same index still serves BM25 queries.
- The `use_hybrid` flag in the request is all that changes — the index is transparent to the caller.

---

## 4. Factory Pattern for Every Service

**Decision:** Each service has a `factory.py` with a `make_*()` function (e.g. `make_opensearch_client()`, `make_database()`, `make_embeddings_service()`).

**Why:**
- Construction logic (reading settings, validating config, calling startup()) is isolated from usage.
- `main.py` only calls `make_*()` functions — it never touches `Settings` directly to construct services.
- Factories can be swapped in tests without changing any service code.
- The factory is the single place to add constructor arguments or feature flags.

---

## 5. `get_settings()` — No Cache in config.py, LRU Cache in dependencies.py

**Decision:** `config.py`'s `get_settings()` creates a fresh `Settings()` each call. `dependencies.py`'s `get_settings()` wraps it with `@lru_cache`.

**Why:** The config module has no FastAPI dependency. Tests can reset the lru_cache without touching config.py. The frozen Pydantic model means settings are immutable once created, so caching is safe.

---

## 6. Graceful Degradation on Optional Services

**Decision:** Redis cache, Jina embeddings, Telegram bot, and Langfuse tracing are all optional. The app starts and serves requests without any of them.

**Why:**
- During early weeks of the course, these services don't exist yet.
- In production, a Redis outage shouldn't take down the search API.
- The dependency return types reflect this: `CacheDep = Annotated[CacheClient | None, ...]` and `TelegramDep = Annotated[Optional[TelegramBot], ...]`.

**Implementation note:** `make_cache_client()` in `main.py` is wrapped in `try/except` so a Redis `ConnectionError` at startup sets `app.state.cache_client = None` rather than aborting. Telegram is handled the same way — if `make_telegram_service()` returns `None` (missing token), startup continues. Any service stored as `None` in `app.state` must be null-checked at usage points.

---

## 7. Multi-Stage Docker Build

**Decision:** Stage 1 uses the official `uv` image to install dependencies; Stage 2 copies only the built venv into `python:3.12-slim`.

**Why:**
- `python:3.12-slim` has no build tools (no `gcc`, no `uv`), so the final image is smaller.
- `UV_COMPILE_BYTECODE=1` pre-compiles `.pyc` files for faster startup.
- `--frozen` ensures the exact locked versions from `uv.lock` are installed — no surprise upgrades in CI.
- `--no-dev` excludes pytest, ruff, mypy, jupyter from the production image.

---

## 8. `database.py` Module-Level Singleton (for Non-FastAPI Contexts)

**Decision:** `src/database.py` uses a module-level `_database` variable as a lazy singleton. This is separate from `dependencies.py`'s database access.

**Why:** Airflow DAGs and standalone scripts cannot use FastAPI's `app.state`. They need a way to get a database session without a running HTTP server. `database.py` provides this path. FastAPI routes never use this file — they use `dependencies.py` instead.

**Note:** This is the one intentional global variable in the codebase. It is isolated to a module that only non-FastAPI code should import.

---

## 9. Two Routers for RAG: `ask_router` and `stream_router` (Same File)

**Decision:** The sync (`/ask`) and streaming (`/stream`) endpoints are defined in the same file but registered as separate routers with different tags.

**Why:** The core logic (`_prepare_chunks_and_sources`) is shared via a private helper function. Splitting into the same file avoids duplication while keeping the router registrations independent. The streaming endpoint returns `StreamingResponse` which FastAPI treats fundamentally differently (can't use `response_model`).

---

## 10. Pydantic Settings with `env_nested_delimiter="__"`

**Decision:** Nested settings are configured with double-underscore delimiter (`REDIS__HOST`, `OPENSEARCH__HOST`) rather than dot notation.

**Why:** Dots are not valid in most shell env var names. Double underscore is the Pydantic standard for nested config. This allows `REDIS__HOST=redis` in Docker Compose to override `settings.redis.host` without changing code.

---

## 11. `upsert()` in PaperRepository Instead of SQL UPSERT

**Decision:** `PaperRepository.upsert()` does `get_by_arxiv_id()` then `create()` or `update()` in Python — not a SQL `INSERT ... ON CONFLICT`.

**Why:** Docling parsing is expensive and papers are fetched by a single Airflow task — true concurrent upsert collisions are unlikely. The application-level approach is simpler to read and debug. A SQL-level `ON CONFLICT DO UPDATE` would be the correct production choice if parallel ingestion workers are added.

---

## 12. `src/middlewares.py` Is a Stub

**Decision:** Request logging functions exist but are not registered as FastAPI middleware.

**Why (inferred from comment in the file):** Week 1 scope was limited to getting services healthy. The comment explicitly notes these are for future `BaseHTTPMiddleware` integration. This is intentional scaffolding — the file exists to show where middleware will go, not to implement it yet.

---

## 13. OpenSearch Security Disabled in Development

**Decision:** `DISABLE_SECURITY_PLUGIN=true` in compose.yml for OpenSearch.

**Why:** Security plugin requires SSL certificates and password configuration that adds friction for learners. This is a development-only stack — TLS and auth would be required before any production deployment.

---

## 14. Langfuse Self-Hosted (Not Cloud)

**Decision:** Full Langfuse v3 stack runs locally in Docker (web, worker, clickhouse, postgres, redis, minio).

**Why:**
- No API keys required for learners.
- Paper content stays private — no data sent to Langfuse cloud.
- Demonstrates that production observability tools can be fully self-hosted.
- Adds 6 containers to the stack, which is the cost of that privacy guarantee.

---

## 15. BM25-First Philosophy

**Decision:** BM25 keyword search is built in Week 3 before vector search in Week 4. The hybrid search system treats BM25 as the foundation, with vectors as enhancement.

**Why (from README):** "Unlike tutorials that jump straight to vector search, we follow the professional path: master keyword search foundations first, then enhance with vectors." BM25 is deterministic, interpretable, and requires no API keys. Many production failures come from over-relying on semantic similarity for queries that are better served by exact keyword matching.

---

## 16. Langfuse v3 SDK Migration (Partially Complete)

**Decision:** `src/services/langfuse/client.py` was refactored from Langfuse v2 to v3 API. The v3 client uses `CallbackHandler` for LangChain/LangGraph auto-tracing and `get_current_trace_id()` for trace ID retrieval.

**Why:** Langfuse v3 changed the recommended integration pattern: instead of manually creating traces and spans, the `CallbackHandler` passed to LangGraph automatically creates the full trace tree. v2's `trace_rag_request()` / `create_span()` are still present in `client.py` for legacy callers (`routers/ask.py`, `services/langfuse/tracer.py`) that haven't been migrated yet.

**Current state:** `trace_langgraph_agent()`, `start_generation()`, `start_span()`, and the `CallbackHandler` approach are v3. `trace_rag_request()` and `create_span()` are v2-style methods retained for backward compatibility. Full v3 migration of the `/ask` and `/stream` endpoints is deferred to Week 6 (observability week).
