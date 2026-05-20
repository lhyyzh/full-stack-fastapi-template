# Full Stack FastAPI Template — Agent Guide

This document provides essential context for AI coding agents working on this project. All information is derived from the actual codebase. When in doubt, trust the source files over this guide.

## Project Overview

This is the official **Full Stack FastAPI Template** from the FastAPI organization (`fastapi/full-stack-fastapi-template`). It is a production-ready template that can be used in three ways:

1. **Fork/clone** and use directly.
2. **Copier** project generation (`copier copy https://github.com/fastapi/full-stack-fastapi-template ...`).
3. It includes `copier.yml` defining input variables (project name, stack name, secret keys, SMTP, DB password, Sentry DSN, etc.) and a post-generation hook at `hooks/post_gen_project.py` that normalizes line endings in shell scripts.

The stack consists of a **FastAPI** backend with **SQLModel** ORM, a **React + TypeScript** frontend with **TanStack Router/Query**, **PostgreSQL** as the database, and **Docker Compose** for development and production orchestration.

## Technology Stack

### Backend
- **Python** >=3.10,<4.0
- **FastAPI** >=0.114.2 (web framework)
- **SQLModel** >=0.0.21 (ORM, SQLAlchemy 2.0 style underneath)
- **Pydantic** >2.0 and **pydantic-settings** >=2.2.1 (data validation, env config)
- **Alembic** >=1.12.1 (database migrations)
- **psycopg** >=3.1.13 (PostgreSQL driver, v3 binary)
- **pyjwt** >=2.8.0 (JWT tokens)
- **pwdlib[argon2,bcrypt]** >=0.3.0 (password hashing, Argon2 primary, Bcrypt legacy)
- **emails** + **jinja2** (email sending and templating)
- **tenacity** (retry logic)
- **sentry-sdk[fastapi]** (error monitoring)
- **Package manager:** `uv` (Astral)

### Frontend
- **React** ^19.1.1 with **TypeScript** ^5.9.3
- **Vite** ^7.3.0 with `@vitejs/plugin-react-swc` (build tool)
- **TanStack Router** (file-based routing)
- **TanStack Query** v5 (server state/cache)
- **React Hook Form** + **Zod** v4 (forms and validation)
- **Axios** (HTTP client, via auto-generated OpenAPI client from `@hey-api/openapi-ts`)
- **Radix UI** primitives + **shadcn/ui** style system
- **Tailwind CSS** v4 (CSS-first configuration, no PostCSS config)
- **Lucide React** + **React Icons**
- **Sonner** (notifications/toasts)
- **Playwright** 1.58.2 (E2E testing)
- **Package manager:** **Bun**

### Infrastructure
- **PostgreSQL** 18 (database)
- **Traefik** 3.6 (reverse proxy / load balancer)
- **Nginx** (serves production frontend build)
- **Mailcatcher** (fake SMTP server for local development)
- **Adminer** (database web admin UI)

## Monorepo Structure

This project uses a **dual-package-manager monorepo**:

- **Python/Backend:** `uv` workspace. Root `pyproject.toml` declares `backend` as a workspace member. Lockfile: `uv.lock`.
- **JavaScript/Frontend:** Bun workspace. Root `package.json` declares `frontend` as a workspace. Lockfile: `bun.lock`.

```
Root (uv workspace + bun workspace)
├── backend/           (FastAPI, SQLModel, Alembic, PostgreSQL, uv)
│   ├── app/           (application code)
│   ├── tests/         (pytest test suite)
│   └── scripts/       (shell automation scripts)
├── frontend/          (React 19, Vite, TanStack, Tailwind, Bun)
│   ├── src/           (source code)
│   └── tests/         (Playwright E2E tests)
├── compose.yml        (production Docker Compose)
├── compose.override.yml   (development overrides)
├── compose.traefik.yml    (standalone Traefik for production servers)
├── .github/workflows/     (CI/CD: 13 workflows)
├── scripts/               (client generation, test helpers)
├── development.md
├── deployment.md
└── .env                   (all env vars and secrets)
```

## Backend Architecture

### Directory Layout
```
backend/app/
├── main.py                    # FastAPI app factory, CORS, Sentry, router inclusion
├── models.py                  # All SQLModel models (User, Item, Token, Message, etc.)
├── crud.py                    # CRUD operations for User and Item
├── utils.py                   # Email templating, sending, password reset tokens
├── initial_data.py            # Script to create first superuser on init
├── backend_pre_start.py       # DB connectivity health check with retries
├── tests_pre_start.py         # Same health check but for test runs
├── core/
│   ├── config.py              # Pydantic Settings (env vars, DB URI, SMTP, etc.)
│   ├── db.py                  # SQLModel engine & init_db()
│   └── security.py            # Password hashing & JWT creation
├── api/
│   ├── main.py                # Aggregate API router
│   ├── deps.py                # FastAPI dependencies: DB session, current user, OAuth2
│   └── routes/
│       ├── login.py           # OAuth2 login, password recovery/reset
│       ├── users.py           # User CRUD, signup, me endpoints
│       ├── items.py           # Item CRUD endpoints
│       ├── utils.py           # Health-check, test-email
│       └── private.py         # Local-only private user creation endpoint
├── alembic/
│   ├── env.py                 # Alembic environment (imports SQLModel metadata)
│   └── versions/              # Migration files
└── email-templates/
    ├── build/                 # Compiled HTML templates
    └── src/                   # MJML source templates
```

### Key Patterns
- **Models:** Single-file pattern in `models.py`. Schema hierarchy: `*Base` → `*Create` / `*Update` / `*Register` → `*` (table) → `*Public`. UUID primary keys, timezone-aware `created_at` fields. `User.items` has `cascade_delete=True`.
- **CRUD:** Thin layer in `crud.py`. `authenticate()` includes a **dummy Argon2 hash** for constant-time comparison to prevent timing attacks. Password hashes auto-upgrade on login (Bcrypt → Argon2) via `verify_and_update`.
- **API Layer:** `deps.py` provides `SessionDep`, `TokenDep`, `CurrentUser`, and `get_current_active_superuser`. Routers are modular and tagged.
- **Config:** `core/config.py` reads from `../.env`. Validates non-default secrets (`changethis` triggers warning/error). Computes `SQLALCHEMY_DATABASE_URI`, `all_cors_origins`, and `emails_enabled`.
- **Private endpoints:** `private.py` router is **only included when `ENVIRONMENT == "local"`**.
- **Email templates:** Built from MJML sources; compiled HTML is rendered via Jinja2.
- **Sentry:** Initialized only when `SENTRY_DSN` is set and environment is not `local`.
- **CORS:** Configured from `BACKEND_CORS_ORIGINS` + `FRONTEND_HOST`.

### Database / Migrations
- **ORM:** SQLModel (SQLAlchemy 2.0 underneath).
- **Database:** PostgreSQL via `psycopg` v3 binary.
- **Engine:** Single global engine in `app/core/db.py`.
- **Sessions:** Dependency-injected generator (`get_db` → `SessionDep`).
- **Migrations:** Alembic with autogenerate support. `target_metadata = SQLModel.metadata` in `alembic/env.py`.
- **Tables are NOT auto-created on startup** (the `SQLModel.metadata.create_all(engine)` line is commented out in `init_db`). Migration via `alembic upgrade head` is required.

### Backend Scripts (`backend/scripts/`)
| Script | Purpose |
|---|---|
| `prestart.sh` | DB retry loop → `alembic upgrade head` → seed superuser |
| `tests-start.sh` | DB retry loop for tests → run `test.sh` |
| `test.sh` | `coverage run -m pytest tests/` → report → HTML |
| `lint.sh` | `mypy app` → `ty check app` → `ruff check app` → `ruff format app --check` |
| `format.sh` | `ruff check app scripts --fix` → `ruff format app scripts` |

## Frontend Architecture

### Directory Layout
```
frontend/src/
├── client/               # Auto-generated OpenAPI client (DO NOT EDIT MANUALLY)
│   ├── core/             # ApiError, OpenAPI config, request helpers
│   ├── sdk.gen.ts        # Generated SDK service classes
│   ├── types.gen.ts      # Generated TypeScript types
│   └── schemas.gen.ts    # Generated JSON schemas
├── components/
│   ├── ui/               # shadcn/ui primitive components (auto-generated)
│   ├── Common/           # Shared layout (AuthLayout, DataTable, Footer, etc.)
│   ├── Sidebar/          # AppSidebar, Main nav, User menu
│   ├── Items/            # Item CRUD components
│   ├── Admin/            # User management CRUD
│   ├── UserSettings/     # ChangePassword, DeleteAccount, UserInformation
│   └── Pending/          # Skeleton loaders for suspense
├── hooks/                # Custom React hooks
│   ├── useAuth.ts        # Authentication logic
│   ├── useCustomToast.ts # Sonner toast wrappers
│   └── useMobile.ts
├── routes/               # File-based routes for TanStack Router
│   ├── __root.tsx        # Root layout with devtools
│   ├── _layout.tsx       # Authenticated layout (sidebar + redirects)
│   ├── _layout/index.tsx # Dashboard (/)
│   ├── _layout/items.tsx # Items page
│   ├── _layout/admin.tsx # Admin users page
│   ├── _layout/settings.tsx # User settings
│   ├── login.tsx
│   ├── signup.tsx
│   ├── recover-password.tsx
│   └── reset-password.tsx
├── lib/
│   └── utils.ts          # cn() helper (clsx + tailwind-merge)
├── main.tsx              # App entry point
├── index.css             # Tailwind v4 CSS with CSS variables and themes
├── utils.ts              # Error handling helpers, getInitials
└── vite-env.d.ts
```

### Key Patterns
- **Routing:** TanStack Router with file-based route generation under `src/routes/`. `_layout.tsx` requires authentication and redirects to `/login` if no `access_token` in `localStorage`. Admin routes have `beforeLoad` guards.
- **API Client:** Fully auto-generated from backend OpenAPI schema via `@hey-api/openapi-ts`. Config in `openapi-ts.config.ts` (legacy/axios + SDK plugin). `OpenAPI.BASE` and `OpenAPI.TOKEN` are configured in `main.tsx` to point at `VITE_API_URL` and read from `localStorage`.
- **State Management:** Server state via TanStack Query. Authentication state encapsulated in `useAuth` hook. Theme via custom `ThemeProvider` (React Context + `localStorage`, dark/light/system, default dark).
- **Styling:** Tailwind CSS v4 with CSS-first configuration in `index.css`. shadcn/ui "New York" style. CSS variables for light/dark modes using `oklch()` colors. `tw-animate-css` for animations.
- **Component variants:** `class-variance-authority` (CVA) used in shadcn components.

### Frontend Scripts (`frontend/package.json`)
| Command | Action |
|---------|--------|
| `bun run dev` | Start Vite dev server (`localhost:5173`) |
| `bun run build` | Type-check + Vite production build → `dist/` |
| `bun run preview` | Preview production build locally |
| `bun run lint` | Biome check with `--write --unsafe` |
| `bun run generate-client` | Regenerate `src/client` from `openapi.json` |
| `bun run test` | `bunx playwright test` |
| `bun run test:ui` | `bunx playwright test --ui` |

## Build and Test Commands

### Full Stack (Docker Compose)
```bash
# Start the entire local development stack with live reload
docker compose watch

# Check logs
docker compose logs
docker compose logs backend
```

### Backend (Local)
```bash
cd backend
uv sync
source .venv/bin/activate

# Run dev server
fastapi dev app/main.py

# Run tests with coverage
bash ./scripts/test.sh

# Run tests inside running Docker stack
docker compose exec backend bash scripts/tests-start.sh

# Run linter checks
bash ./scripts/lint.sh

# Auto-format
bash ./scripts/format.sh

# Run migrations inside container
docker compose exec backend bash
alembic revision --autogenerate -m "Description"
alembic upgrade head
```

### Frontend (Local)
```bash
cd frontend
bun install
bun run dev

# Run E2E tests (requires Docker Compose stack running)
docker compose up -d --wait backend
bunx playwright test
bunx playwright test --ui

# Generate API client from running backend
bash ./scripts/generate-client.sh
```

### Root-level Scripts
```bash
# Root package.json proxies frontend scripts
bun run dev      # → bun run --filter frontend dev
bun run lint     # → bun run --filter frontend lint
bun run test     # → bun run --filter frontend test
bun run test:ui  # → bun run --filter frontend test:ui
```

## Code Style Guidelines

### Python (Backend)
- **Formatter/Linter:** `ruff` (targets Python 3.10). Config in `backend/pyproject.toml`.
- **Type checker:** `mypy` in **strict mode**, plus `ty` (error-on-warning).
- **Ruff selects:** E, W, F, I, B, C4, UP, ARG001, T201.
- **Ruff ignores:** E501 (line too long), B008, W191, B904.
- **Excluded from linting/type-checking:** `alembic/` directory.
- **Coverage:** Tracks `app` source. HTML report at `backend/htmlcov/index.html`. CI enforces **90% coverage threshold**.
- **No `print` statements allowed** (T201 rule).

### TypeScript (Frontend)
- **Formatter/Linter:** **Biome** v2.3. Config in `frontend/biome.json`.
- **Biome exclusions:** `dist/`, `node_modules/`, `src/routeTree.gen.ts`, `src/client/**/*`, `src/components/ui/**/*`, `playwright-report/`, `playwright.config.ts`.
- **Biome rules:** `recommended` enabled. `suspicious.noExplicitAny: off`, `suspicious.noArrayIndexKey: off`, `style.noNonNullAssertion: off`, `style.noParameterAssign: error`, `style.useSelfClosingElements: error`, `style.noUselessElse: error`.
- **Formatter:** spaces, double quotes, semicolons `asNeeded`.
- **TypeScript:** strict mode. `noUnusedLocals`, `noUnusedParameters` enabled.

### General
- Pre-commit hooks are managed by **prek** (modern alternative to pre-commit), not classic `pre-commit`.
- Install prek: `uv run prek install -f` (from inside `backend/` directory).
- Run all hooks manually: `uv run prek run --all-files`.

## Testing Instructions

### Backend Tests
- **Framework:** `pytest` with `coverage`.
- **Entry point:** `bash ./scripts/test.sh` (from `backend/`).
- **Fixtures (`tests/conftest.py`):**
  - `db` (session, autouse): initializes DB, cleans `Item` + `User` tables after all tests.
  - `client` (module): `TestClient` wrapping the FastAPI app.
  - `superuser_token_headers` (module): auth headers for default superuser.
  - `normal_user_token_headers` (module): auth headers for test user (`settings.EMAIL_TEST_USER`).
- **Utilities (`tests/utils/`):** `random_email()`, `random_lower_string()`, `get_superuser_token_headers()`, `user_authentication_headers()`, `create_random_user()`, `authentication_token_from_email()`.
- **Coverage report:** `backend/htmlcov/index.html`.

### Frontend E2E Tests
- **Framework:** Playwright v1.58.2.
- **Config:** `playwright.config.ts`. Base URL `http://localhost:5173`.
- **Auth setup:** `auth.setup.ts` logs in as the first superuser and saves state to `playwright/.auth/user.json`. The `chromium` project depends on `setup` and reuses that storage state.
- **Web server:** Playwright config automatically runs `bun run dev` before tests.
- **Workers:** 1 on CI, parallel locally. `retries: 2` on CI. `fullyParallel: true`.
- **Test files:** `login.spec.ts`, `sign-up.spec.ts`, `reset-password.spec.ts`, `items.spec.ts`, `admin.spec.ts`, `user-settings.spec.ts`.
- **Docker E2E service:** `compose.override.yml` includes a `playwright` service for CI E2E testing.

## Docker Compose Services

### Production (`compose.yml`)
| Service | Image | Purpose |
|---------|-------|---------|
| `db` | `postgres:18` | PostgreSQL database |
| `adminer` | `adminer` | DB admin UI (dark theme) |
| `prestart` | Backend image | Runs Alembic migrations and seeds initial data before backend starts |
| `backend` | Backend image | FastAPI app on port 8000, healthchecked at `/api/v1/utils/health-check/` |
| `frontend` | Frontend image | Nginx serving static React app on port 80 |

All production-exposed services use Traefik labels on the `traefik-public` external network.

### Development Overrides (`compose.override.yml`)
| Service | Additions |
|---------|-----------|
| `proxy` | Traefik 3.6 with debug dashboard, HTTP entrypoint on 80, dashboard on 8090 |
| `db` | Exposes port `5432:5432` |
| `adminer` | Exposes port `8080:8080` |
| `backend` | Exposes port `8000:8000`, `fastapi run --reload`, Docker Compose Watch for live sync, SMTP routed to mailcatcher |
| `mailcatcher` | `schickling/mailcatcher`, ports `1080:1080` and `1025:1025` |
| `frontend` | Exposes port `5173:80`, `VITE_API_URL=http://localhost:8000` |
| `playwright` | Playwright E2E test runner image |

### Development URLs
- Frontend: `http://localhost:5173`
- Backend: `http://localhost:8000`
- Swagger UI: `http://localhost:8000/docs`
- ReDoc: `http://localhost:8000/redoc`
- Adminer: `http://localhost:8080`
- Traefik UI: `http://localhost:8090`
- MailCatcher: `http://localhost:1080`

With `DOMAIN=localhost.tiangolo.com`:
- Frontend: `http://dashboard.localhost.tiangolo.com`
- Backend: `http://api.localhost.tiangolo.com`

## Security Considerations

- **Secrets validation:** `core/config.py` warns/errors if secrets still have the default value `changethis`. You MUST change `SECRET_KEY`, `FIRST_SUPERUSER_PASSWORD`, and `POSTGRES_PASSWORD` before deploying.
- **Password hashing:** Argon2 is primary; Bcrypt is supported for legacy hashes. Hashes auto-upgrade on successful login.
- **Timing attack prevention:** `authenticate()` in `crud.py` performs a dummy Argon2 hash verification even when the user does not exist, to prevent username enumeration via timing.
- **JWT:** Access tokens are short-lived. Password reset tokens are JWT-based with expiration.
- **Generic responses:** Password recovery endpoints return generic success messages regardless of whether the email exists, to prevent user enumeration.
- **CORS:** Strictly configured via `BACKEND_CORS_ORIGINS`. Do not use `*` in production.
- **Private endpoints:** The `private.py` router (local-only user creation) is **only mounted when `ENVIRONMENT == "local"`**. Never expose it in production.
- **Dependency guard:** GitHub Actions workflow `guard-dependencies.yml` auto-closes PRs modifying `pyproject.toml`/`uv.lock` from non-org members.
- **Security audit:** `zizmor.yml` runs `zizmor` to audit GitHub Actions workflows for security issues.

## Deployment Process

This project expects a **public Traefik proxy** to handle HTTPS certificates and routing.

### Manual Deployment
1. Set up a remote server with Docker Engine.
2. Configure DNS and wildcard subdomain (e.g., `*.fastapi-project.example.com`).
3. Copy `compose.traefik.yml` to the server and start public Traefik:
   ```bash
   docker network create traefik-public
   docker compose -f compose.traefik.yml up -d
   ```
4. Set required environment variables (`ENVIRONMENT`, `DOMAIN`, `SECRET_KEY`, `POSTGRES_PASSWORD`, `FIRST_SUPERUSER_PASSWORD`, `BACKEND_CORS_ORIGINS`, etc.).
5. Deploy the stack:
   ```bash
   docker compose -f compose.yml build
   docker compose -f compose.yml up -d
   ```
   Note: explicitly use `compose.yml` (not `compose.override.yml`) for production.

### Continuous Deployment (GitHub Actions)
- **Staging:** triggers on push to `master`. Uses self-hosted runner labeled `staging`.
- **Production:** triggers on release publish. Uses self-hosted runner labeled `production`.
- Both workflows use GitHub Environments for scoped secrets and deployment protection rules.
- Required environment secrets: `DOMAIN_PRODUCTION`, `DOMAIN_STAGING`, `STACK_NAME_PRODUCTION`, `STACK_NAME_STAGING`, `EMAILS_FROM_EMAIL`, `FIRST_SUPERUSER`, `FIRST_SUPERUSER_PASSWORD`, `POSTGRES_PASSWORD`, `SECRET_KEY`, `LATEST_CHANGES`, `SMOKESHOW_AUTH_KEY`.

### Production URLs
Replace `fastapi-project.example.com` with your domain:
- Frontend: `https://dashboard.fastapi-project.example.com`
- Backend API: `https://api.fastapi-project.example.com`
- Adminer: `https://adminer.fastapi-project.example.com`
- Traefik UI: `https://traefik.fastapi-project.example.com`

## CI/CD Workflows (`.github/workflows/`)

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `test-backend.yml` | PR/push to `master` | Spins up DB + mailcatcher, runs migrations, pytest with coverage, enforces 90% threshold |
| `test-docker-compose.yml` | PR/push to `master` | Builds full stack, starts it, curls health-check endpoints |
| `playwright.yml` | PR/push to `master` | Runs E2E tests in 4 sharded containers, merges blob reports |
| `pre-commit.yml` | PR/push | Runs `prek` with ruff, biome, etc. Auto-commits fixes |
| `zizmor.yml` | PR/push | Security audit of GitHub Actions workflows |
| `deploy-staging.yml` | Push to `master` | Deploys to staging self-hosted runner |
| `deploy-production.yml` | Release publish | Deploys to production self-hosted runner |
| `guard-dependencies.yml` | `pull_request_target` | Auto-closes dependency PRs from non-org members |
| `labeler.yml` | PRs | Auto-labels and enforces required labels |
| `add-to-project.yml` | Issues/PRs | Adds to fastapi GitHub project board |
| `issue-manager.yml` | Scheduled | Auto-closes stale issues/PRs |
| `detect-conflicts.yml` | PRs | Labels PRs with merge conflicts |
| `latest-changes.yml` | PR merge to `master` | Auto-updates `release-notes.md` |
| `smokeshow.yml` | After `Test Backend` | Uploads backend HTML coverage report as Smokeshow site |

## Environment Variables (`.env`)

The `.env` file at the project root contains all configuration. Key variables include:

- `PROJECT_NAME`, `STACK_NAME`
- `DOMAIN`, `ENVIRONMENT`
- `SECRET_KEY` (must not be `changethis` in production)
- `BACKEND_CORS_ORIGINS`
- `FIRST_SUPERUSER`, `FIRST_SUPERUSER_PASSWORD`
- `SMTP_HOST`, `SMTP_USER`, `SMTP_PASSWORD`, `EMAILS_FROM_EMAIL`
- `POSTGRES_SERVER`, `POSTGRES_PORT`, `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB`
- `SENTRY_DSN`
- `FRONTEND_HOST`

## Important Conventions

1. **Never manually edit `frontend/src/client/`**. This directory is auto-generated by `@hey-api/openapi-ts` from the backend OpenAPI schema. Run `bash ./scripts/generate-client.sh` after backend API changes.
2. **Never manually edit `frontend/src/components/ui/`**. These are shadcn/ui primitives. If you need to add a new primitive, use the shadcn/ui CLI or copy pattern.
3. **Always create Alembic migrations when changing SQLModel models.** Do not rely on `SQLModel.metadata.create_all(engine)` (it is commented out). Run `alembic revision --autogenerate -m "..."` inside the backend container.
4. **Always compile MJML to HTML** when editing email templates. Source lives in `backend/app/email-templates/src/`; compiled output goes to `backend/app/email-templates/build/`.
5. **Frontend routes are file-based.** Adding a new page means adding a new file under `frontend/src/routes/`. The route tree is auto-generated; do not manually edit `routeTree.gen.ts`.
6. **The `playwright` service in `compose.override.yml` uses `ipc: host`** for browser sandboxing.
7. **Root `package.json` and `pyproject.toml` are workspace manifests.** The actual application configs live in `frontend/package.json` and `backend/pyproject.toml`.
