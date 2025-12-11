This file gives focused, repo-specific guidance for AI coding agents (Copilot-style) so they can be productive quickly.

High-level summary
- Monorepo: SvelteKit frontend (root `src/`) + FastAPI backend (`backend/open_webui/`).
- Frontend served by Vite; backend is FastAPI + Uvicorn. Realtime features use Socket.IO (see `backend/open_webui/socket`).
- Models/runners are pluggable: Ollama, OpenAI-compatible endpoints, and a built-in pipelines integration. Runtime behavior is controlled heavily by environment variables (see `backend/open_webui/env.py` and `backend/open_webui/config.py`).

What to read first (fast path)
- `README.md` — install and Docker examples.
- `backend/open_webui/env.py` — canonical env var names, DATA_DIR, DATABASE_URL, REDIS_URL, feature toggles (ENABLE_*).
- `backend/open_webui/config.py` — PersistentConfig pattern and runtime config migration/mgmt.
- `backend/open_webui/main.py` — where routers, sockets, and middleware are wired together.
- `package.json` — frontend dev/build/test/lint scripts.

Quick dev workflows (concrete commands)
- Frontend dev: in repo root run `npm run dev` (this runs `pyodide:fetch` then `vite dev`).
- Backend dev: run `backend/dev.sh` (starts uvicorn with --reload). You can also run `uvicorn open_webui.main:app --reload` from `backend/` — `dev.sh` shows exact envs.
- Full local stack: `docker-compose up --build` (see `docker-compose.yaml`) or use the Docker examples in `README.md` (OLLAMA_BASE_URL, DATA_DIR volumes).
- Build frontend for production: `npm run build` then ensure `FRONTEND_BUILD_DIR` points to the build output expected by backend (env `FRONTEND_BUILD_DIR`).

Testing and linting
- Frontend unit tests: `npm run test:frontend` (vitest).
- E2E: Cypress / Playwright configs are present — `npm run cy:open` for Cypress GUI.
- Lint frontend: `npm run lint:frontend`; lint backend with `pylint backend/` (top-level `npm run lint` runs both).

Key integration patterns and conventions
- Environment-first configuration: Many runtime behaviors are toggled via env vars (search for ENABLE_, *_BASE_URL, DATA_DIR, DATABASE_URL, REDIS_URL in `env.py` and `config.py`). Prefer using the existing `PersistentConfig` pattern when introducing settings that should persist to DB.
- PersistentConfig: `backend/open_webui/config.py` migrates a `data/config.json` into the DB and exposes a `PersistentConfig` wrapper that prefers DB-stored values when ENABLE_PERSISTENT_CONFIG is true. Use this for config that can be edited at runtime.
- Routers: Add HTTP APIs under `backend/open_webui/routers/` and register them in `main.py` alongside existing routers (chats, models, retrieval, etc.). Follow request/response shapes used by nearby routers.
- Socket/Realtime: The Socket.IO server is in `backend/open_webui/socket`; event emitter and model usage helpers are exposed and used by `main.py`.
- Retrieval / RAG: Retrieval code (embeddings, reranking) lives under `backend/open_webui/routers/retrieval` — changing retrieval implies touching embedding configuration, RAG_TEMPLATE, and possibly the vector DB clients listed in `pyproject.toml` (chroma, qdrant, etc.).

Why things look like they do (design intent)
- Extensibility: many domain areas are separated into routers and models so new model runners, adapters, or endpoints can be added without touching core request-handling logic.
- Runtime configurability: persistent DB-backed config allows the running instance to be reconfigured via the UI; env vars provide defaults and feature toggles.
- Offline and multi-runner focus: the codebase supports local/offline runners (Ollama, built-in engines) and cloud/OpenAI-compatible endpoints — expect a mix of direct HTTP clients and wrapper adapters.

Where to make common edits (examples)
- Add a new API: create `backend/open_webui/routers/myfeature.py` and import/register it in `backend/open_webui/main.py`.
- Add a new persistent config: create a `PersistentConfig('NAME','path.in.config', default)` in `config.py` or `env.py` style and use `AppConfig` if cross-process Redis propagation is needed.
- Wire a new frontend page: add a Svelte route under `src/routes/` and call backend endpoints under `/api` or the registered router paths.

Important files & directories (quick reference)
- backend/open_webui/              — backend package (routers, models, socket, utils)
- backend/dev.sh                   — backend dev start script (uvicorn --reload)
- backend/open_webui/env.py        — authoritative env vars and DATA_DIR, DB, REDIS defaults
- backend/open_webui/config.py     — persistent config, migrations are run on import
- backend/open_webui/main.py       — app + router wiring
- src/                             — SvelteKit frontend (Vite, pyodide fetch script usage referenced in `package.json`)
- package.json                     — frontend scripts: dev, build, lint, test
- docker-compose.yaml / Dockerfile — containerized deployment examples (OLLAMA, open-webui services)

Agent-specific rules (short)
- Prefer editing existing routers and follow the same request/response patterns — examine `chats`, `models`, or `retrieval` for examples.
- When introducing env-configured behavior, add a `PersistentConfig` if the value must be editable via UI/config DB.
- Avoid changing database schema without running/adjusting the Alembic migration pipeline — `config.py` runs migrations on import; create Alembic revisions when necessary.

If anything in this file is unclear or you want me to expand examples (e.g., show a tiny router skeleton and tests), tell me which area to expand and I'll iterate.
