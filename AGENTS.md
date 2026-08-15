# AGENTS.md

Guidance for AI coding agents (Claude Code, etc.) working in this repository.

> Note: this repo's root `.gitignore` explicitly excludes `CLAUDE.md` (a
> deliberate maintainer choice — see line 18). This `AGENTS.md` file exists
> so agent guidance is still tracked in git; it is not ignored.

## What This Is

**Open WebUI** is a full-stack, self-hostable AI chat platform: a Python
(FastAPI) backend and a SvelteKit frontend, shipped together as one app
(the built frontend is served by the backend in production). It connects to
Ollama and any OpenAI-compatible API, and layers on RAG, RBAC, a plugin
system (Filters/Actions/Pipes/Tools/Skills), notes, channels, calendar,
automations, and enterprise auth (LDAP, OAuth/SSO, SCIM).

## Layout

| Path | Purpose |
|------|---------|
| `backend/open_webui/main.py` | FastAPI app: middleware stack, router registration, static frontend mounting. |
| `backend/open_webui/routers/` | One module per API surface (`auths`, `chats`, `models`, `knowledge`, `tools`, `skills`, `functions`, `retrieval`, `automations`, `calendar`, `scim`, ...), mounted under `/api/v1/<name>` in `main.py`. `ollama.py`/`openai.py` are mounted at `/ollama`, `/openai`. |
| `backend/open_webui/models/` | SQLAlchemy ORM models + Pydantic schemas, one file per domain entity (mirrors `routers/`). |
| `backend/open_webui/internal/db.py` | Engine/session setup — supports SQLite (incl. `sqlite+sqlcipher://` for encryption), PostgreSQL, and reads `DATABASE_URL`. |
| `backend/open_webui/migrations/` | Alembic migrations (`versions/`, ~48 revisions at time of writing). |
| `backend/open_webui/retrieval/` | RAG stack: `loaders/` (content extraction engines), `vector/` (9 supported vector DBs), `web/` (web-search providers), `external.py`. |
| `backend/open_webui/utils/` | Cross-cutting helpers: `auth.py`, `oauth.py`, `chat.py`, `middleware.py`, `mcp/` (MCP client support), `plugin.py` (dynamic Function loading — see Conventions), `access_control/`, `telemetry/` (OpenTelemetry), `redis.py`. |
| `backend/open_webui/tools/` | Built-in tool implementations (`builtin.py`, `knowledge_fs.py`). |
| `backend/open_webui/socket/` | Socket.IO (`python-socketio`) real-time layer. |
| `backend/open_webui/config.py`, `env.py`, `constants.py` | Runtime configuration (DB-persisted + env-var driven), low-level env parsing, shared string constants. |
| `backend/open_webui/data/` | Default persisted data dir (uploads, `webui.db` for SQLite, etc.) in a running instance. |
| `src/routes/` | SvelteKit routes. `(app)` is the authenticated app shell (`admin/`, `c/` chat, `workspace/`, `notes/`, `channels/`, `calendar/`, `automations/`, `playground/`); `auth/` is the login/signup flow; `s/`, `watch/` are share/embed routes. |
| `src/lib/apis/` | Frontend API client, one module per backend router (`chats/`, `models/`, `knowledge/`, `tools/`, ...) — thin `fetch` wrappers around `/api/v1/...`. `apis/index.ts` holds shared/legacy endpoints. |
| `src/lib/components/` | Svelte components, organized by area (`admin/`, `chat/`, `workspace/`, `channel/`, `notes/`, `calendar/`, `automations/`, `common/`, `icons/`, `layout/`). |
| `src/lib/stores/index.ts` | Global Svelte stores (writable stores for user/session/settings/models/etc.) — the frontend's shared client-side state. |
| `src/lib/i18n/` | i18next resources; locale JSON lives under `src/lib/i18n/locales/`. |
| `src/lib/workers/`, `src/lib/pyodide/` | Web Worker code and the in-browser Python (Pyodide) code-interpreter integration. |
| `static/` | Static assets served as-is. |
| `docs/SECURITY.md` | Security policy (points to GitHub's private vulnerability reporting — see README). |
| `scripts/` | Node helper scripts, e.g. `prepare-pyodide.js` (fetches Pyodide runtime before dev/build). |

## Commands

### Frontend (root `package.json`, npm; Node `>=18.13.0 <=22.x.x`)

```bash
npm install
npm run dev              # pyodide:fetch + vite dev --host  (http://localhost:5173)
npm run dev:5050         # same, on port 5050
npm run build            # pyodide:fetch + vite build -> build/
npm run preview          # vite preview of a production build

npm run check            # svelte-kit sync + svelte-check (type checking)
npm run lint             # lint:frontend (eslint --fix) + lint:types (check) + lint:backend (pylint)
npm run lint:frontend    # eslint . --fix
npm run format           # prettier --write "**/*.{js,ts,svelte,css,md,html,json}"
npm run i18n:parse       # i18next-parser + prettier over src/lib/i18n

npm run test:frontend    # vitest --passWithNoTests  (no vitest spec files are checked into this repo yet)
npm run cy:open          # cypress open (no cypress/ test directory is checked into this repo)
```

### Backend (`pyproject.toml`, uv; Python `>=3.11,<3.13`)

```bash
cd backend
pip install -r requirements.txt      # or: uv sync (uses uv.lock)
bash dev.sh                          # uvicorn --reload on :8080, CORS open to :5173/:8080
# or directly:
uvicorn open_webui.main:app --port 8080 --reload

ruff format . --exclude .venv --exclude venv     # formatting (also: npm run format:backend from root)
ruff check --select=F --ignore=F401,F403,F405,F541,F811,F841 .   # the check CI actually runs
pylint backend/                                   # npm run lint:backend from root
```

No pytest/unit-test suite is checked into this repo (`pyproject.toml`'s `all`
extra lists `pytest`/`pytest-docker`/`playwright` as optional dependencies,
and CI (`.github/workflows/backend.yaml`) only runs `ruff format --check` and
a `ruff check --select=F` logic-error pass — there is no backend test job).

### Full stack

Run backend (`bash backend/dev.sh`, port 8080) and frontend (`npm run dev`,
port 5173) side by side; Vite's dev server proxies API calls, and
`CORS_ALLOW_ORIGIN` in `dev.sh` is pre-set for this split. In production the
backend serves the built frontend directly from one origin/port (see
`FRONTEND_BUILD_DIR` in `main.py` and the `force-include` build hook in
`pyproject.toml` that packages `build/` as `open_webui/frontend`).

### Docker

```bash
docker compose up -d              # docker-compose.yaml (+ .gpu/.amdgpu/.api/.data/.otel/.playwright overlays)
make install / make start / make stop / make update   # Makefile wraps docker compose
```

## Architecture & Conventions

- **One router + one model module per domain entity.** Adding a new API
  surface means adding both `backend/open_webui/routers/<name>.py` (FastAPI
  `APIRouter`, registered in `main.py` under `/api/v1/<name>`) and
  `backend/open_webui/models/<name>.py` (SQLAlchemy model + Pydantic
  request/response schemas) — the two directories mirror each other 1:1 for
  nearly every entity (`chats`, `models`, `knowledge`, `tools`, `skills`,
  `functions`, `folders`, `groups`, `prompts`, `notes`, `channels`,
  `calendar`, `automations`, ...).
- **Schema changes go through Alembic.** `backend/open_webui/migrations/`
  holds the migration history; don't hand-edit the DB schema — add a new
  revision. `internal/db.py` supports SQLite (plain or `sqlite+sqlcipher://`
  for at-rest encryption), PostgreSQL, and reads `DATABASE_URL`.
- **The plugin system is dynamic Python loading, not a fixed registry.**
  `utils/plugin.py` loads user-authored "Functions" (Filters, Actions, Pipes
  — see the `type` column on the `Function` model in `models/functions.py`)
  from source stored in the DB, each with an optional `Valves` /
  `UserValves` Pydantic class for configurable settings (including
  dynamically-resolved dropdown options via `resolve_valves_schema_options`).
  Tools and Skills follow a similar pattern; `tools/builtin.py` holds the
  first-party built-ins, `utils/mcp/` bridges to external MCP servers, and
  `retrieval/` similarly is a family of pluggable loaders/vector-DB/web-search
  backends selected by config rather than hardcoded.
- **Config is layered: env vars seed DB-persisted settings.** `config.py`
  defines the persisted/admin-editable configuration surface (backed by the
  `config` table, with `import_legacy_config_json`/`seed_registered_defaults`
  for migration/bootstrap); `env.py` holds lower-level, process-start-time
  env var parsing (ports, feature toggles, secrets) that isn't meant to be
  changed at runtime. When adding a new setting, decide which layer it
  belongs to rather than defaulting to `env.py`.
- **Frontend API modules mirror backend routers.** Each `src/lib/apis/<name>/`
  directory is a thin `fetch`-based client for the matching
  `/api/v1/<name>` router; add new backend endpoints and their frontend
  caller in the same PR, following an existing sibling module's shape
  (see `src/lib/apis/index.ts` for the older/shared and OpenAI/tool-server
  endpoints).
- **Global client state lives in `src/lib/stores/index.ts`** as plain Svelte
  writable stores (user session, settings, model list, socket connection,
  etc.) — components read/write these rather than re-fetching ad hoc.
- **SvelteKit route groups encode auth/layout boundaries**: everything under
  `src/routes/(app)/` assumes an authenticated session and the main app
  shell/sidebar; `src/routes/auth/` is unauthenticated; `s/` and `watch/` are
  standalone share/embed views without the app shell.
- **The frontend build needs Pyodide fetched first.** `npm run dev`/`build`
  both run `scripts/prepare-pyodide.js` (`pyodide:fetch`) before Vite — this
  powers the in-browser Python code interpreter (`src/lib/pyodide/`,
  `src/lib/workers/`). Don't bypass this step when scripting a build.
- **Version is derived from `package.json`, not duplicated.** `pyproject.toml`
  uses a hatch `[tool.hatch.version]` regex hook to read the version straight
  out of `package.json`'s `"version"` field — bump it in one place.

## Testing

- **No test suites are currently checked into this repo.** `git ls-files`
  turns up no `*.test.*`/`*.spec.*` frontend specs and no `test_*.py`/
  `conftest.py` backend tests, despite `npm run test:frontend` (vitest),
  `npm run cy:open` (Cypress), and pytest/Playwright appearing as available
  tooling (`vitest`, `cypress` in `package.json`; `pytest`, `pytest-docker`,
  `playwright` in `pyproject.toml`'s `all` extra). `test/test_files/` holds
  only fixture data (e.g. `sd-empty.pt`), not tests.
- **CI does not run either test runner.** `.github/workflows/frontend.yaml`
  runs format check, i18n-string verification, a production build, and
  `npm run test:frontend` (which passes vacuously — `--passWithNoTests`) on
  every push/PR to `main`/`dev`. `.github/workflows/backend.yaml` runs only
  `ruff format --check` and a narrow `ruff check --select=F` pass across
  Python 3.11/3.12 — no pytest step exists.
- If you add tests, place frontend specs alongside source (Vitest picks up
  `*.test.ts`/`*.spec.ts` by default) or under a `cypress/` directory for E2E
  (referenced by `cy:open` and `docker-compose.playwright.yaml` but not
  present), and backend tests under a `test/` or `backend/open_webui/`-mirrored
  path discoverable by pytest — there's no existing convention to match yet,
  so pick one consistent with the CI workflow you also update.
- Per the PR template (see below), **manual testing with screenshots is the
  actual expectation** for this project, not automated coverage.

## Gotchas

- **PRs traditionally target `dev`, not `main`**, in the upstream
  `open-webui/open-webui` repo — its PR template explicitly says PRs against
  `main` are auto-closed. This fork currently only has a `main` branch (no
  `dev`), so that specific rule doesn't apply here as-is; if a `dev` branch
  is ever added to this fork, check the template again before opening PRs
  against `main`.
- **`.gitignore` excludes `CLAUDE.md` on purpose** (see line 18) — that's why
  this file is named `AGENTS.md` instead. Don't try to "fix" this by editing
  `.gitignore`.
- **Lint/format tooling exists at two disabled tiers.**
  `.github/workflows/lint-backend.disabled` and `lint-frontend.disabled` are
  present but inactive (`.disabled` suffix), as is `codespell.disabled` — the
  active CI is only `backend.yaml` (ruff format + narrow ruff check) and
  `frontend.yaml` (format/i18n/build/vitest). Don't assume pylint, full ESLint,
  or codespell are enforced in CI even though `npm run lint` wires them up
  locally.
- **`backend.yaml` CI only checks formatting, not full lint** — `ruff check`
  in CI is restricted to `--select=F` (pyflakes) with several codes ignored
  (`F401,F403,F405,F541,F811,F841`); the broader `[tool.ruff.lint]` `select`
  list in `pyproject.toml` (E, W, I, UP, C90, Q, ICN) is what `pylint`/local
  `ruff check` would enforce, but CI doesn't gate on it.
- **Two dependency manifests must stay compatible**: `backend/requirements.txt`
  (and `requirements-min.txt`) vs. `pyproject.toml`'s `dependencies` — check
  both when adding/bumping a Python dependency, and note version-pin comments
  in `pyproject.toml` that exist for a reason (e.g. `aiohttp==3.13.5 # do not
  update to 3.13.3 - broken`, the Playwright version comment tying it to
  `docker-compose.playwright.yaml`).
- **`npm install` in CI uses `--force`** (`frontend.yaml`) — there's a known
  peer-dependency conflict somewhere in the tree; don't be surprised if a
  plain `npm install` warns/fails locally without `--force`.
- **No root `CONTRIBUTING.md`** exists in this checkout; contribution rules
  live in `.github/pull_request_template.md` (CLA checkbox required, PR-prefix
  convention: `feat`/`fix`/`docs`/`chore`/`refactor`/`perf`/`style`/`test`/
  `i18n`/`build`/`ci`/`BREAKING CHANGE`/`WIP`) and `CODE_OF_CONDUCT.md`.
- **License is not plain MIT/Apache** — see `LICENSE` + `LICENSE_HISTORY`;
  redistributions must preserve "Open WebUI" branding per the project's
  license terms. Be careful with anything that touches branding/naming.
- **Multiple `docker-compose.*.yaml` overlays** exist for GPU (`.gpu`,
  `.amdgpu`), API-only (`.api`), data-volume-only (`.data`), OpenTelemetry
  (`.otel`), Playwright (`.playwright`), and a manual A1111 integration test
  (`.a1111-test`) — compose them with `-f docker-compose.yaml -f
  docker-compose.<overlay>.yaml`, don't assume one file covers everything.
