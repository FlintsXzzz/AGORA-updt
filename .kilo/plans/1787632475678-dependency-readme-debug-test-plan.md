# Plan: Fix Dependencies, README, Debugging & Testing — AGORA WhatsApp AI Accountant

## Context
Repo is a FastAPI + LangChain/LangGraph backend (Python 3.10) for a WhatsApp AI accountant. Four tasks requested: (1) fix the broken dependency set, (2) generate a README, (3) debug runtime bugs, (4) add a test workflow.

Sandbox has **no pip/network**, so the dependency fix is produced by analysis and validated by the user running `pip install` in their own environment. No code is mutated in this plan.

---

## Task 1 — Fix dependencies (`requirements.txt`)

### Root cause
`langchain-google-genai==1.0.6` depends on `langchain-core>=0.3.0` (and `google-genai>=1.0.0`). The file pins `langchain-core==0.2.9` and `langchain==0.2.5` (which constrains core to `<0.3`). The resolver cannot satisfy both → `ResolutionImpossible`. `langgraph==0.1.5` is also pre-0.2 and misaligned.

### Recommended fix (upgrade path — keeps `google-genai` usage in `Engine.py`/`tasks.py`)
Replace the LangChain block with a self-consistent 0.3 set:

```
# LangChain / LangGraph (AI Orchestration) — aligned to langchain 0.3 + langgraph 0.2
langchain==0.3.20
langchain-core==0.3.20
langchain-community==0.3.20
langchain-google-genai==1.0.6
langgraph==0.2.7
```

Keep the rest unchanged:
```
fastapi==0.111.0
uvicorn[standard]==0.30.1
python-multipart==0.0.9
sqlalchemy==2.0.30
psycopg2-binary==2.9.9
alembic==1.13.1
pgvector==0.2.5
google-genai>=1.0.0
python-dotenv==1.0.1
httpx==0.27.0
pydantic==2.7.4
```

### Validation (user, in their env)
```
python -m venv .venv && . .venv/bin/activate
pip install -r requirements.txt
pip check                     # must report no conflicts
python -c "import fastapi, langchain, langgraph, langchain_google_genai, google.genai; print('ok')"
```
If a specific patch version above is unavailable, loosen to `langchain~=0.3.0`, `langchain-core~=0.3.0`, `langchain-community~=0.3.0`, `langgraph>=0.2,<0.3` and re-run.
Alternative strategy (downgrade google-genai instead) is NOT recommended: it would require also installing the legacy `google-generativeai` package and splitting the SDK the code depends on.

---

## Task 2 — Debugging (code fixes)

1. **`models.User` missing `role` column** (`src/models.py`)
   Migration `76b0ee887c06` creates `users.role`, but `models.User` omits it. This breaks `setup_db.py:169` (`User(..., role="owner")`) and `Engine.py` `register_user` (`User(..., role=payload.role)`) with `TypeError`, and `list_tenant_users` (`u.role`) with `AttributeError`.
   Fix: add `role = Column(String, nullable=True)` to `User` in `src/models.py` so model matches migration + API responses. (Matches commit intent of keeping per-user role data without RBAC enforcement.)

2. **Alembic `sys.path` wrong** (`migrations/alembic/env.py:13`)
   It appends `migrations/` (one level too shallow). `from database import Base` / `import models` live in `src/`. Fix:
   ```python
   REPO_ROOT = os.path.dirname(os.path.dirname(os.path.dirname(__file__)))
   sys.path.append(os.path.join(REPO_ROOT, "src"))
   ```

3. **`setup_db.py` runs alembic from wrong dir**
   It calls `alembic upgrade head` with `cwd=src`, but `alembic.ini` is in `migrations/`. Alembic fails to find config and silently falls back to `create_all()`. Fix the run command to:
   ```
   alembic -c migrations/alembic.ini upgrade head
   ```
   (with `PYTHONPATH=src`). Update `setup_db.py:run_migrations` accordingly, or set `PYTHONPATH=src` before the subprocess.

4. **Placeholder Gemini model IDs (runtime, not import)**
   - `src/agent.py:79` `ChatGoogleGenerativeAI(model="gemini-3.5-flash")`
   - `src/rag.py:12` `GoogleGenerativeAIEmbeddings(model="models/gemini-embedding-2")`
   - `src/Engine.py:441,467` and `src/tasks.py:119` `model="gemini-3.5-flash"`
   `gemini-3.5-flash` and `gemini-embedding-2` are not real model IDs → 404 at runtime. Replace with valid IDs the user's key can access (e.g. `gemini-2.5-flash` / `gemini-2.0-flash` and `text-embedding-004` or `gemini-embedding-001`). Define as env-overridable constants to avoid hardcoding.

5. **Run/import model note** (document, not strictly fix)
   Modules use top-level imports (`from tools import ...`, `from database import ...`), so the app must run with `src` on `PYTHONPATH`: `PYTHONPATH=src uvicorn Engine:app --host 0.0.0.0 --port 8000`. Document this in README.

---

## Task 3 — README (`README.md`)

Generate `README.md` (Indonesian + English summary) covering:
- Project overview (from `deskripsi_projek.md`)
- Architecture diagram (FastAPI engine / LangGraph agent / PGVector RAG / WhatsApp Cloud API)
- Prerequisites (Python 3.10+, Postgres or Supabase, pgvector)
- Setup: `cp .env.example .env`, fill keys, `pip install -r requirements.txt`
- Run app: `PYTHONPATH=src uvicorn Engine:app --reload --port 8000`
- DB setup: `PYTHONPATH=src python src/setup_db.py` (or `PYTHONPATH=src alembic -c migrations/alembic.ini upgrade head`)
- Docker: `docker compose up -d` for local Postgres
- Environment variable reference (table from `.env.example`)
- API endpoint list (from `Engine.py` docstring)
- Testing instructions (Task 4)
- Notes on placeholder model IDs (Task 2.4)

---

## Task 4 — Testing workflow

Add `tests/` (run with `pytest`, using SQLite so no Postgres needed):

1. **Pure-function unit tests** (no DB/network):
   - `tests/test_normalize.py`: `normalize_numeric_token`, `normalize_integer_token` (IDR formats: `"1.000.000"`, `"1,5 jt"`, `"500rb"`).
   - `tests/test_ocr.py`: `validate_ocr_output`, `fallback_parse_items` with sample receipt text/JSON.

2. **Endpoint/integration tests** (`tests/test_api.py`) using FastAPI `TestClient` + `SQLite` in-memory DB:
   - Override `get_db` dependency to a SQLite `SessionLocal`.
   - `POST /transactions` → `GET /dashboard/summary` asserts income/expense/net math.
   - `GET /dashboard/transactions` pagination/filtering.
   - `POST /users` and `GET /tenants/{id}/users` (validates `role` column fix).
   - Health `GET /`.
   - Add `conftest.py` to build SQLite engine + tables and inject `PYTHONPATH=src` (or import via `src.` prefix).

3. **Agent smoke test** (`tests/test_agent.py`):
   - Monkeypatch `agent.get_model` to return a stub `ChatGoogleGenerativeAI` that emits a fixed tool call, then assert `process_message` returns expected `reply`/`requires_clarification`. Avoids real LLM/API calls.

4. **Add to README**: `pip install pytest` and `PYTHONPATH=src pytest -q`.

---

## Risks / open questions
- Exact LangChain patch versions may differ by release date; `pip install` + `pip check` is the source of truth.
- Gemini model IDs depend on the user's API access tier — confirm valid IDs before changing placeholders.
- `pgvector` Python package is separate from `langchain_community` PGVector; both are required and currently both listed.
- No `role` enforcement (RBAC removed per git history) — column kept for data only.

## Validation (definition of done)
- `pip install -r requirements.txt` succeeds; `pip check` clean.
- App imports: `PYTHONPATH=src python -c "import Engine, agent, tools, tasks, rag"`.
- `setup_db.py` seeds without `TypeError`; `GET /tenants/default-tenant/users` returns `role`.
- `PYTHONPATH=src pytest -q` passes.
- `README.md` present and matches actual commands.
