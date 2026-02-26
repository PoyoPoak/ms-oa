# Copilot Coding Agent Instructions

Trust these instructions first. Only search the codebase if information here is incomplete or found to be in error.

## Project Identity

**MSOA** is a personal interview showcase project — an end-to-end MLOps web application for monitoring synthetic pacemaker telemetry data. It is built on top of the FastAPI Full Stack Template but is **not** a template under development. All work done in this repository is for building a specific application.

**Purpose**: Demonstrate skills in data engineering, AI/ML, CI/CD, and automation during a software engineering interview at a medical device R&D company. The project must show mastery of: data pipelines, ML model training and evaluation, CI/CD automation (targeting Azure DevOps), and a React dashboard that visualizes model results.

**Application Domain**: Synthetic pacemaker telemetry. The ML model (Random Forest) predicts whether a pacemaker will fail in the next 7 days based on device sensor readings. A FastAPI backend ingests new telemetry, retrains the model, and pushes metrics to a React dashboard showing patient-level risk.

## Repository Overview

Full-stack web application built on the FastAPI Full Stack Template. **This is not a template under development—it is a template being used to build a real web application.** Make changes as application features, not template improvements.

- **Backend**: Python 3.10+ with FastAPI, SQLModel ORM, PostgreSQL, Alembic migrations, JWT auth
- **Frontend**: React 19 + TypeScript + Vite 7 + Tailwind CSS 4 + shadcn/ui + TanStack Router/Query
- **Package managers**: `uv` (Python), `bun` (JavaScript/TypeScript)
- **Infrastructure**: Docker Compose with Traefik reverse proxy, Mailcatcher for dev email
- **Linting**: Ruff + mypy (backend), Biome (frontend), prek pre-commit hooks
- **Testing**: pytest (backend), Playwright (frontend E2E)
- **CI/CD target**: Azure DevOps (GitHub Actions is used during development; Azure DevOps pipelines will be added for the final MLOps automation)

## Project Layout

```
.                           # Root: workspace configs, Docker Compose, .env
├── backend/
│   ├── app/
│   │   ├── main.py         # FastAPI app entrypoint
│   │   ├── models.py       # SQLModel data models (User, Item, etc.)
│   │   ├── crud.py         # Database CRUD operations
│   │   ├── core/
│   │   │   ├── config.py   # Pydantic Settings (reads ../.env)
│   │   │   ├── db.py       # Database engine & init
│   │   │   └── security.py # JWT & password hashing
│   │   ├── api/
│   │   │   ├── main.py     # API router aggregation
│   │   │   ├── deps.py     # Dependency injection
│   │   │   └── routes/     # Endpoint modules (items, users, login, utils, private)
│   │   ├── alembic/        # Migration scripts
│   │   └── email-templates/# MJML source (src/) and built HTML (build/)
│   ├── tests/              # pytest tests (conftest.py, api/, crud/, utils/)
│   ├── scripts/            # prestart.sh, tests-start.sh, test.sh, lint.sh, format.sh
│   ├── pyproject.toml      # Python deps, ruff/mypy/coverage config
│   ├── alembic.ini         # Alembic configuration
│   └── Dockerfile          # Python 3.10 + uv
├── frontend/
│   ├── src/
│   │   ├── main.tsx        # React entrypoint
│   │   ├── client/         # Auto-generated OpenAPI SDK (DO NOT edit manually)
│   │   ├── components/     # App components + ui/ (shadcn, DO NOT edit ui/ manually)
│   │   ├── routes/         # TanStack Router pages
│   │   └── hooks/          # Custom React hooks
│   ├── tests/              # Playwright E2E tests
│   ├── package.json        # Bun deps & scripts
│   ├── biome.json          # Biome linter/formatter config
│   ├── playwright.config.ts
│   ├── vite.config.ts
│   ├── openapi-ts.config.ts# OpenAPI client generation config
│   ├── components.json     # shadcn/ui configuration
│   ├── Dockerfile          # Bun build → Nginx
│   └── Dockerfile.playwright
├── scripts/                # Root-level scripts
│   ├── generate-client.sh  # Regenerate frontend OpenAPI SDK from backend
│   ├── test.sh             # Full Docker-based test run
│   └── test-local.sh       # Legacy Docker Compose test
├── openspec/               # OpenSpec spec-driven change workflow
│   └── config.yaml         # Project context and purpose
├── .env                    # Environment variables (shared by backend & Docker Compose)
├── compose.yml             # Production Docker Compose
├── compose.override.yml    # Dev overrides (ports, hot-reload, mailcatcher, playwright)
├── pyproject.toml          # Root uv workspace config (members: ["backend"])
├── package.json            # Root bun workspace config (workspaces: ["frontend"])
└── .pre-commit-config.yaml # prek/pre-commit hooks config
```

## Application Domain — Pacemaker Telemetry MLOps

The core product is an MLOps pipeline and monitoring dashboard for synthetic pacemaker data.

### Telemetry Data Schema

Each record has these fields (stored as approximately 2,000,000 rows in the initial dataset; grows as new data is ingested):

| Field | Type | Description |
|---|---|---|
| `Patient_ID` | int | Patient identifier (excluded from model training; used for grouping/display) |
| `Timestamp` | unix int | Collection time (excluded from model training; ensures chronological order) |
| `Lead_Impedance_Ohms` | float (2 dp) | Wire lead resistance. Normal ≈ 500 Ω; upward drift → lead fracture/degradation |
| `Capture_Threshold_V` | float (2 dp) | Min voltage to stimulate heart. Normal ≈ 0.70 V; rise → lead losing tissue contact |
| `R_Wave_Sensing_mV` | float (2 dp) | R-wave voltage detected. Normal ≈ 12.00 mV; drop → sensing degradation |
| `Battery_Voltage_V` | float (2 dp) | Battery health. Healthy start ≈ 2.95 V; < 2.60 V → critical depletion |
| `Target_Fail_Next_7d` | bool (0/1) | Target label. `1` = device predicted to fail within 7 days (engineered from clinical logs) |

### ML Pipeline

- **Algorithm**: Random Forest classifier (scikit-learn)
- **Features used in training**: `Lead_Impedance_Ohms`, `Capture_Threshold_V`, `R_Wave_Sensing_mV`, `Battery_Voltage_V`
- **Target**: `Target_Fail_Next_7d`
- **Key metrics shown on dashboard**: Accuracy, Precision, Recall, F1 score
- **Data generation**: `util/generate_data.py` (planned; will create the synthetic dataset — `util/` directory to be created at root level)
- **Retraining trigger**: POST to a FastAPI endpoint that adds new data, retrains, evaluates, then publishes updated metrics/predictions

### Planned MLOps Flow (in progress)

```
Incoming data (POST /api/v1/telemetry/ingest)
  → Append to dataset
  → Retrain Random Forest model
  → Evaluate model (accuracy, precision, recall, F1)
  → Persist trained model artifact
  → Update dashboard with new metrics + patient-level risk predictions
```

The MLOps pipeline will also run on a scheduled Azure DevOps pipeline (daily/weekly) to keep the model fresh.

### Frontend Dashboard Goal

The React dashboard will display:
- Model performance metrics (latest training run)
- A list/table of patients currently in the **danger zone** (`Target_Fail_Next_7d = 1`)
- Historical trend of model accuracy over retraining runs

> **Current state**: The application still contains the template `Item` model and placeholder CRUD routes from the original FastAPI Full Stack Template. These will be removed/replaced with pacemaker domain models (`TelemetryRecord`, `ModelRun`, etc.) as the project progresses.

## Build, Test & Lint Commands

### Backend

All backend commands run from the **repository root** unless noted. The backend requires a running PostgreSQL database (use Docker Compose).

```bash
# Install Python dependencies (from repo root)
uv sync --all-packages

# Start database + mailcatcher
docker compose up -d db mailcatcher

# Run Alembic migrations + seed data (required before tests)
cd backend && uv run bash scripts/prestart.sh && cd ..

# Run backend tests with coverage (requires db + migrations)
cd backend && uv run bash scripts/tests-start.sh && cd ..

# Coverage must remain ≥90% — CI will fail otherwise
cd backend && uv run coverage report --fail-under=90 && cd ..

# Lint backend (mypy + ruff check + ruff format check)
cd backend && uv run bash scripts/lint.sh && cd ..

# Auto-format backend
cd backend && uv run bash scripts/format.sh && cd ..

# Ruff only (used by pre-commit)
uv run ruff check --force-exclude --fix
uv run ruff format --force-exclude
```

### Frontend

```bash
# Install frontend dependencies (from repo root)
bun install

# Lint + format frontend (Biome — auto-fixes)
bun run lint

# Build frontend (TypeScript check + Vite build)
cd frontend && bun run build && cd ..

# Run Playwright E2E tests (requires full stack running)
docker compose up -d --wait backend
cd frontend && bunx playwright test && cd ..
```

### Regenerating the Frontend OpenAPI Client

Always regenerate after changing any backend API endpoint, model, or route:

```bash
bash ./scripts/generate-client.sh
```

This script: extracts OpenAPI JSON from the backend → writes `frontend/openapi.json` → runs `openapi-ts` → runs `bun run lint`. The generated files in `frontend/src/client/` must be committed.

### Full Stack Docker

```bash
# Start everything for development (hot-reload enabled)
docker compose watch

# Or start specific services
docker compose up -d db mailcatcher backend frontend

# Clean teardown
docker compose down -v --remove-orphans
```

## CI Checks on Pull Requests

Every PR triggers these workflows that **must all pass**:

| Workflow | File | What it checks |
|---|---|---|
| **pre-commit** | `pre-commit.yml` | Runs `prek` (ruff check, ruff format, biome check, large files, YAML/TOML, trailing whitespace, end-of-file fixer, frontend SDK regeneration). Auto-commits fixes. |
| **Test Backend** | `test-backend.yml` | Spins up PostgreSQL + mailcatcher via Docker Compose, runs `uv run bash scripts/prestart.sh` then `uv run bash scripts/tests-start.sh`, enforces ≥90% coverage. Python 3.10. |
| **Playwright Tests** | `playwright.yml` | Builds Docker images, runs Playwright tests sharded across 4 workers. Only runs when backend/, frontend/, .env, compose*.yml, or the workflow file changes. |
| **Test Docker Compose** | `test-docker-compose.yml` | Builds and starts full stack, curls health endpoints. |

> **Note**: There is no automated labeler or conflict-detector workflow in this repository. Labels are not enforced by CI.

### Known CI Behavior — `action_required` Status

Workflows that require repository secrets (`POSTGRES_PASSWORD`, `SECRET_KEY`, `FIRST_SUPERUSER`, `FIRST_SUPERUSER_PASSWORD`) will show `action_required` status on PRs from first-time contributors or bots. This is a GitHub Actions environment protection feature, **not a code failure**. The repository owner must manually approve these workflow runs via the GitHub Actions UI. Once approved, the workflows run normally.

**Workaround**: The owner must navigate to the Actions tab and approve pending workflow runs on new PRs.

### Pre-commit Hooks (prek)

The `.pre-commit-config.yaml` defines these hooks executed in order:

1. `check-added-large-files` — blocks large file commits
2. `check-toml` / `check-yaml` — syntax validation
3. `end-of-file-fixer` — ensures files end with newline (excludes `frontend/src/client/`, `backend/app/email-templates/build/`)
4. `trailing-whitespace` — removes trailing spaces (excludes `frontend/src/client/`)
5. `biome check` — frontend linting (runs `npm run lint` on `frontend/` files)
6. `ruff check` — Python linting with auto-fix
7. `ruff format` — Python formatting
8. `generate-frontend-sdk` — re-generates `frontend/src/client/` when backend files change

## Workflow Protocols

### CI .env Handling

Workflow jobs must not rely on a committed `.env`. Any job that runs `docker compose` must create a `.env` during the workflow run (typically by copying `.env-template` and overriding required keys with GitHub Actions secrets). Local development still uses the developer's `.env` file. Github Actions secrets and variables will always be made as secrets, never as variables, to prevent accidental exposure in logs.

### Creating Alembic Migrations Protocol
**Critical Workflow When Modifying Models:**
1.  **Modify** `SQLModel` classes in `backend/app/models.py`.
2.  **Verify Imports**: Ensure the module defining your models is imported by Alembic. In this repo, `backend/app/alembic/env.py` already imports `app.models`, so new models in that file are picked up automatically. If you add models in a new module, import that module in `env.py`.
3.  **Generate**: Run `docker compose exec backend alembic revision --autogenerate -m "message"`.
4.  **Verify**: Check the generated file in `backend/app/alembic/versions/` to ensure `upgrade()` contains actual SQL changes.
5.  **Apply**: `docker compose exec backend alembic upgrade head`.

### Cross-Stack Dependency Protocol

If you modify `backend/app/api/` or `backend/app/models.py`:

1. If a backend dev server is running, restart it (or let hot-reload pick up changes).
2. Run `bash ./scripts/generate-client.sh` to regenerate `frontend/src/client/`.
3. Commit the generated changes in `frontend/src/client/`.
4. Fix any resulting type errors in `frontend/src/`.

### Playwright Agent Protocol

When asked to write or fix E2E tests, adopt the Planner/Generator/Healer pattern:

- **Planner**: Before writing code, analyze the `frontend/` logic and propose a test plan in Markdown (e.g., `frontend/tests/specs/checkout.md`). Create the `frontend/tests/specs/` folder if needed.
- **Generator**: Write tests using semantic locators only (role, label, text, test id).
	- GOOD: `page.getByRole('button', { name: 'Submit' })`
	- BAD: `page.locator('div > .btn-primary')`
- **Healer**: If a test fails in CI, assume the UI changed. Analyze the HTML dump and propose an updated locator rather than rewriting the logic.
- **Seeding/Auth**: Use the Playwright setup project in `frontend/tests/auth.setup.ts` and its storage state at `playwright/.auth/user.json` for authenticated tests. Do not script full login UI flows in every test unless the test is explicitly about login behavior.

### OpenSpec Workflow (spec-driven changes)

This repository uses an **OpenSpec** spec-driven workflow via the skills in `.agents/skills/`. When implementing new features, prefer using the OpenSpec flow:

1. **Explore** (`openspec-explore` skill): Think through the problem before writing code.
2. **Propose** (`openspec-propose` skill): Generate a complete change proposal (design doc, spec, tasks).
3. **Apply** (`openspec-apply-change` skill): Implement tasks from the spec.
4. **Archive** (`openspec-archive-change` skill): Finalize and archive after implementation.

The project context lives in `openspec/config.yaml` — always consult it when making architectural decisions or scoping new features.

## Key Conventions

- **Python**: Target Python 3.10. Ruff enforces pycodestyle, pyflakes, isort, flake8-bugbear, comprehensions, pyupgrade, no-print, no-unused-args. No `print()` statements allowed (T201 rule).
- **TypeScript**: Biome enforces double quotes, space indentation, semicolons only as needed. Files in `src/client/`, `src/components/ui/`, and `src/routeTree.gen.ts` are excluded from linting (auto-generated).
- **Do not manually edit**: `frontend/src/client/**` (generated by openapi-ts), `frontend/src/components/ui/**` (managed by shadcn/ui), `frontend/src/routeTree.gen.ts` (generated by TanStack Router plugin).
- **Environment variables**: All in root `.env` file. Backend reads via `pydantic-settings` from `../.env`. Never commit real secrets.
- **API prefix**: All backend routes are under `/api/v1`. The `private` router only loads when `ENVIRONMENT=local`.
- **Database models**: Defined in `backend/app/models.py` using SQLModel. Always create Alembic migrations after model changes.
- **Shell scripts**: Must use LF line endings (enforced by `.gitattributes`).
- **Coverage threshold**: Backend test coverage must be ≥90%.
- **ML dependencies**: Use `scikit-learn` for ML. Add to `backend/pyproject.toml` dependencies; install with `uv sync`.

## Common Pitfalls

- **Backend tests require a running PostgreSQL**: Always `docker compose up -d db mailcatcher` and run `prestart.sh` before running pytest.
- **Frontend client out of sync**: If you change backend API routes/models and forget to run `bash ./scripts/generate-client.sh`, pre-commit and CI will fail.
- **`uv sync --all-packages`** must be run from the repo root (not `backend/`) to resolve the workspace correctly. Running `uv sync` from `backend/` also works for backend-only deps.
- **`bun install`** must be run from the repo root to install the workspace. Running from `frontend/` also works but the root lockfile (`bun.lock`) is the source of truth.
- **PR label requirement**: CI does NOT enforce labels in this repository. The labeler workflow does not exist here.
- **CI `action_required`**: Workflows that require secrets will show `action_required` for first-time bot PRs. This is not a code failure — the repo owner must manually approve via the GitHub Actions UI.
- **Template Items model**: The `Item` model/routes/tests are still from the original template. They will remain until explicitly replaced with MSOA domain models. Do not delete them unless directed to do so.

# Boundaries
- 🚫 NEVER use `pip`, `poetry`, `npm`, or `yarn`.
- ✅ ALWAYS use `uv` for Python and `bun` for JavaScript/TypeScript.
- 🚫 NEVER edit `uv.lock` or `bun.lock` manually.
- 🚫 NEVER remove or rename the `Item` model/routes without explicit instruction — it is placeholder content being kept intentionally.
- ✅ When adding ML/data science code, place it under `backend/app/` following existing module conventions.