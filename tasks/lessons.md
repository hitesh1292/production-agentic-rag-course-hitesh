# Lessons Log

Updated after every correction. Read at the start of each session.

---

## L001 — Always run ruff --fix before committing

**Pattern:** Unsorted import blocks accumulate silently across multiple files (dependencies.py, agents/, telegram/) and only surface when `ruff check` is run.

**Rule:** After any code change — even small ones — run `uv run ruff check src/ --fix` before marking the task done. Do not run `ruff check` without `--fix` first, then check again to confirm zero errors.

**Why:** Ruff's import sorter (I001) is auto-fixable and takes <1 second. There is no reason to leave these unfixed. A clean lint pass is a hard requirement before done.

---

## L002 — The health endpoint is /api/v1/health, not /health

**Pattern:** Written `/health` in verify.md and CLAUDE.md commands — the actual FastAPI route registered in `main.py` and `ping.py` is `/api/v1/health`.

**Rule:** Always use `curl localhost:8000/api/v1/health`. The bare `/health` path does not exist and returns 404. Double-check router prefixes in `main.py` before writing any URL in docs or scripts.

---

## L003 — Never put heavy service imports at the top level of Airflow DAG task modules

**Root cause:** `airflow/dags/arxiv_ingestion/common.py` and `indexing.py` had `from src.services.*` imports at module level. Airflow's DAG file processor re-imports every DAG file on every scheduler heartbeat. Those imports triggered `docling` → `torch` loading, which takes 30+ seconds and exceeds Airflow's DAG import timeout, marking the DAG as broken.

**Fix:** Move all heavy `src.services.*` imports inside the function bodies (`get_cached_services()`, `index_papers_hybrid()`, `verify_hybrid_index()`). Python's import system caches modules after the first load, so the cost is paid only once at actual task execution time — not at DAG parse time.

**Rule:** In any Airflow DAG task module, `from src.services.*` imports must live inside functions, never at module level. Add a comment at the top of the file as a guard: `# Heavy imports are deferred inside functions — do NOT move to module level.`

---

## L002 — The health endpoint is /api/v1/health, not /health

**Pattern:** Written `/health` in verify.md and CLAUDE.md commands — the actual FastAPI route registered in `main.py` and `ping.py` is `/api/v1/health`.

**Rule:** Always use `curl localhost:8000/api/v1/health`. The bare `/health` path does not exist and returns 404. Double-check router prefixes in `main.py` before writing any URL in docs or scripts.
