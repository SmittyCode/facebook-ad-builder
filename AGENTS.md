# AGENTS.md

Project guidance for Codex and other coding agents working in this repository.

## Project overview

Facebook Ad Builder is a full-stack application for competitor research, AI ad generation, ad remixing, Facebook campaign management, and performance reporting.

Technology:

- Frontend: React 19, Vite, TailwindCSS, Vitest, ESLint
- Backend: FastAPI, Python 3.11+, SQLAlchemy, Alembic, pytest
- Database: PostgreSQL; SQLite is deprecated and must not be introduced as a fallback
- Storage: Cloudflare R2, with local `uploads/` fallback for development
- AI services: Google Gemini and Fal.ai
- Hosting: Railway

## Working agreements

- Inspect the relevant files and existing patterns before editing.
- Keep changes focused and preserve unrelated user work.
- Never commit secrets, tokens, credentials, or `.env.local` files.
- Do not run destructive database, deployment, or repository operations without explicit approval.
- Do not remove or reorganize unrelated tracked media under `backend/uploads/`.
- Prefer existing dependencies and patterns before adding new ones.
- Update tests and documentation when behavior changes.
- Use `ast-grep` for large structural JavaScript/JSX refactors when it is available.

## Startup checks

For end-to-end browser testing, check whether `agent-browser` is installed:

```bash
command -v agent-browser >/dev/null || echo "WARNING: agent-browser is not installed"
```

Do not install tools automatically; report missing prerequisites or ask before installing them.

## Development commands

### Backend

```bash
cd backend
python -m venv venv
source venv/bin/activate  # Windows: venv\\Scripts\\activate
pip install -r requirements.txt
python init_db.py
uvicorn app.main:app --reload --port 8000
pytest
pytest tests/test_<feature>.py -v
```

The backend API is normally available at `http://localhost:8000`, with docs at `http://localhost:8000/api/v1/docs`.

### Frontend

```bash
cd frontend
npm install
npm run dev
npm run build
npm run lint
npm run test:unit
npm run test:smoke
```

The frontend is normally available at `http://localhost:5173`. Smoke tests require a running app and the configured test environment.

### Browser smoke tests

```bash
cd frontend
BASE_URL=https://your-app.com npm run test:smoke
BASE_URL=https://your-app.com npm run test:login
TEST_EMAIL=user@example.com TEST_PASSWORD=xxx npm run test:auth
```

Use `agent-browser` only when it is installed and the target environment is authorized.

## Architecture

### Backend

All API routes use the `/api/v1` prefix. Database access uses FastAPI dependency injection with `Depends(get_db)`.

Important locations:

```text
backend/app/main.py                 FastAPI app, CORS, router registration
backend/app/database.py             SQLAlchemy engine and session setup
backend/app/models.py               SQLAlchemy models
backend/app/core/config.py          Environment configuration and validation
backend/app/api/v1/                  API route modules
backend/app/services/                Business logic and external services
backend/app/schemas/                 Pydantic request/response schemas
backend/alembic/                     Database migrations
backend/tests/                       Backend tests
```

Core entities include Brand, Product, CustomerProfile, WinningAd, GeneratedAd, FacebookCampaign, FacebookAdSet, FacebookAd, and ScrapedAd.

The Facebook integration uses the `facebook-business` SDK. AI integrations use Gemini and Fal.ai. File uploads use Cloudflare R2 when configured and local storage otherwise.

### Frontend

```text
frontend/src/App.jsx                Router and top-level providers
frontend/src/main.jsx               Application entry point
frontend/src/components/            Reusable UI and wizard components
frontend/src/pages/                 Page-level route components
frontend/src/context/               Toast, brand, and campaign state
frontend/src/lib/                   Supabase and Facebook API helpers
frontend/src/__tests__/             Frontend test setup and tests
```

API calls should use the configured API URL, normally `VITE_API_URL` or the local backend at `http://localhost:8000/api/v1`.

## Mandatory UI rules

Never use browser `alert()` or `confirm()`.

Use `useToast` from `frontend/src/context/ToastContext.jsx`:

```javascript
const { showSuccess, showError, showWarning, showInfo } = useToast();
```

Use custom confirmation modals for destructive actions. Modals should have a clear title and description, a blurred semi-transparent backdrop, an appropriately styled destructive button, and an icon indicating the action type.

## Database and environment

PostgreSQL is required. Do not silently substitute SQLite.

Create a local environment file from the example and supply credentials through environment variables. Important variables include:

- `DATABASE_URL`
- `SECRET_KEY`
- `R2_ACCOUNT_ID`, `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`, `R2_BUCKET_NAME`, `R2_PUBLIC_URL`
- `GEMINI_API_KEY`, `FAL_AI_API_KEY`, and any other configured AI keys
- Facebook Marketing API credentials and account identifiers
- `VITE_API_URL` and frontend Facebook configuration as applicable

Treat the local and production Railway database and R2 storage as shared external state. Prefer mocks or isolated test data for tests, and never print credentials in command output or logs.

Backend tests require a reachable PostgreSQL test database through `DATABASE_URL` or `TEST_DATABASE_URL`; they must not be pointed at production. The test fixtures create tables and clean up selected test records, so use a dedicated test database.

## Code style

### Python

- Follow PEP 8.
- Use Black-compatible formatting with an 88-character line length.
- Use Ruff or Flake8 conventions where configured.
- Use `snake_case` for functions and variables and `PascalCase` for classes.

### JavaScript and React

- Follow the repository ESLint and Prettier conventions.
- Use `PascalCase` for components, `camelCase` for functions and variables, and `UPPER_SNAKE_CASE` for constants.
- Reuse existing context, toast, modal, and API helper patterns.

## Security and deployment

- Preserve CORS restrictions and update CSP configuration when adding approved origins.
- Keep JWT authentication and refresh-token behavior intact unless the task explicitly changes authentication.
- Preserve upload validation and the 10 MB image limit.
- Store secrets only in environment variables.
- Railway uses one config-as-code file per service. The root `railway.toml` configures the backend Docker service; `frontend/railway.toml` configures the frontend Railpack service.
- Railway backend service settings must use root directory `/` and config path `/railway.toml`; frontend settings must use root directory `/frontend` and config path `/frontend/railway.toml`.
- Do not reintroduce a `[[services]]` multi-service structure into `railway.toml`.
- Database migrations run as part of backend deployment; initial role/permission seeding is handled separately by `backend/init_db.py` when required.
- After deploying a feature, run applicable backend, frontend, and smoke tests before declaring it complete.

## Useful feature areas

The main product areas are brand management, product catalog, customer profiles, competitor research, AI ad generation, ad remixing, Facebook campaign management, generated-ad galleries, and reporting.

Consult `specifications.md`, `README.md`, and the relevant tests before changing behavior in these areas.

## Common gotchas

- `DATABASE_URL` must be PostgreSQL.
- Frontend API configuration is build-time configuration.
- When adding an allowed frontend origin, update both backend CORS and the frontend CSP.
- Facebook ad account IDs may be normalized with an `act_` prefix in the Facebook service.
- Commit new Alembic migration files together with the code that requires them.

## Current validation baseline

The repository was audited on 2026-08-22:

- From `frontend/`, `npm run build` passes.
- From `frontend/`, `npm run lint` currently reports existing lint failures; fix relevant failures when modifying nearby code rather than assuming lint is clean.
- From `frontend/`, `npm run test:unit` currently exits because no frontend test files match the Vitest include pattern.
- Backend Python compilation passes with a writable `PYTHONPYCACHEPREFIX`.
- Backend pytest collection passes and discovers 100 tests, but execution requires a reachable PostgreSQL test database.
- The project requires Python 3.11+; Python 3.9 is not a representative supported runtime.
