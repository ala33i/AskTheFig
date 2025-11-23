## Quick Context

This repo is a fork/pack of Open WebUI. Key pieces:
- Frontend: Svelte + Vite in `src/` — run with `npm run dev` and build with `npm run build`.
- Backend: FastAPI app under `backend/open_webui/` (main entry `main.py`). The backend exposes REST routes in `backend/open_webui/routers/` and a socket server in `backend/open_webui/socket/`.
- Packaging: Python project using `pyproject.toml` (Hatch/rye) and JS tooling via `package.json`.
- Deploy: Dockerfile + `run.sh` and `docker-compose.yaml` (services: `open-webui`, `ollama`).

## What matters to an AI coding agent (immediately actionable)

- To start the frontend dev server: `npm install` then `npm run dev` (root `package.json`).
- To run locally as a full stack (recommended for integration work): `docker compose up --build` or use `./run.sh` for a single container.
- Backend entrypoint is `backend/open_webui/main.py` — new routers must be imported and registered there (see the large `from open_webui.routers import ...` block).
- Config and env: `backend/open_webui/config.py` reads most runtime options from environment via `backend/open_webui/env.py`. Add env vars to `docker-compose.yaml` or call `export` locally when running the backend.
- DB & migrations: Alembic migrations live under `backend/open_webui/migrations/`. On startup `config.run_migrations()` is invoked to auto-apply migrations — avoid directly duplicating migration logic.

## Project-specific patterns & conventions

- Persistent runtime configuration: the app migrates `data/config.json` into a DB table and exposes `PersistentConfig` wrappers (`config.py`). Changes to config should use `PersistentConfig.save()` or update via the UI database path, not by editing `config.json` directly.
- Routers follow `backend/open_webui/routers/<name>.py` and are imported en masse in `main.py`. Add a router and then add it to that import list.
- DB models are in `backend/open_webui/models/` and use SQLAlchemy; sessions come from `backend/open_webui/internal/db.py`.
- Websockets/events: look at `backend/open_webui/socket/main.py` for event emitter and background tasks (usage cleanup, model usage metrics).
- Frontend ↔ Backend: frontend calls backend REST APIs under the same origin (or via host.docker.internal in Docker). See `src/lib/apis/` and `src/routes/` for concrete examples of API usage and how auth tokens are passed.

## Common developer workflows (explicit commands)

- Frontend dev: `npm install && npm run dev` (open on host with `--host`).
- Frontend build: `npm run build` then the backend serves `FRONTEND_BUILD_DIR` when present.
- Full stack (local Docker): `docker compose up --build` or `./run.sh` (single container flow uses `Dockerfile`).
- Lint/format: `npm run lint` (runs frontend ESLint + `pylint backend/`) and `npm run format:backend` (Black).
- Tests: frontend unit tests via `npm run test:frontend` (Vitest). E2E tests with Cypress are in `cypress/` (open with `npm run cy:open`). Backend tests are run via pytest if configured in your environment (see `pyproject.toml` optional dev deps).

## Integration points to watch

- Ollama: `OLLAMA_BASE_URL` (used heavily for model inference). Default is in `docker-compose.yaml` pointing to the `ollama` service.
- OpenAI / Google / Anthropic: configured via env vars handled in `config.py` and used throughout routers (images, audio, RAG).
- Redis: sessions use `starsessions` and a Redis connection is configured via `REDIS_URL` / sentinel options in `env.py` / `config.py`.
- Image tools: AUTOMATIC1111 and COMFYUI endpoints appear in `config.py`; if adding image backends, follow these existing env patterns.

## Safe edits and gotchas

- Do not bypass `PersistentConfig` persistence when changing runtime config — use the config API or `PersistentConfig.save()`.
- Adding a new router requires: create file in `routers/`, implement endpoints, import it in `main.py` import block so it is registered.
- The app auto-runs alembic migrations on startup (see `config.run_migrations()`). If you edit migrations, validate them locally with a disposable DB first.

## Files worth opening first

- `backend/open_webui/main.py` — app wiring and router imports.
- `backend/open_webui/config.py` and `backend/open_webui/env.py` — configuration and env var mapping.
- `backend/open_webui/routers/` — API examples.
- `backend/open_webui/socket/` — real-time and background task patterns.
- `src/` and `package.json` — frontend patterns and build scripts.
- `docker-compose.yaml`, `run.sh`, `Dockerfile` — how services are wired for local/full-stack runs.

If anything here is unclear or you want the agent to include extra detail (examples of adding a new router, or a checklist for creating migrations), tell me which part to expand and I will update this file accordingly.
