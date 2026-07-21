# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Skills

This project includes custom slash commands for common workflows:

| Skill | Description |
| ----- | ----------- |
| `/new-model` | Full CRUD scaffolding — model → migration → routes → frontend SDK → pages |
| `/client-sync` | Regenerate TypeScript API client from backend OpenAPI schema |
| `/db-migrate` | Create and apply Alembic database migrations |

## Rules

Project-specific conventions are in `.claude/rules/`:

- [api-conventions.md](.claude/rules/api-conventions.md) — Permission patterns, response types, error handling, CRUD conventions
- [frontend-patterns.md](.claude/rules/frontend-patterns.md) — SDK usage, form handling, styling, route structure
- [db-conventions.md](.claude/rules/db-conventions.md) — Migration naming, auto-generation caveats, rollback procedures

## Common Commands

### Development

```bash
# Start full stack with Docker Compose (recommended for first run)
docker compose watch

# Frontend dev server only (faster iteration, needs backend running)
bun run dev                 # from project root
cd frontend && bun run dev  # same, directly

# Backend dev server only (needs PostgreSQL running)
cd backend && fastapi dev app/main.py

# Run backend tests (stack must be running)
docker compose exec backend bash scripts/tests-start.sh

# Run backend tests with extra pytest args
docker compose exec backend bash scripts/tests-start.sh -x -k "test_name"

# Run frontend E2E tests (stack must be running, backend must be healthy)
bunx playwright test
bunx playwright test --ui

# Install dependencies
bun install          # frontend (root, uses workspaces)
cd backend && uv sync # backend Python deps
```

### Code Quality

```bash
# Run all pre-commit hooks manually
cd backend && uv run prek run --all-files

# Python linting/formatting only
cd backend && uv run ruff check --fix backend/app
cd backend && uv run ruff format backend/app
cd backend && uv run mypy backend/app

# Frontend linting
bun run lint     # biome check (from root)
```

### Database Migrations & Client SDK

```bash
# Enter the backend container (run alembic commands inside)
docker compose exec backend bash

# Regenerate the TypeScript API client from backend OpenAPI schema
bash ./scripts/generate-client.sh
```

Use `/db-migrate` for the full migration workflow. Use `/client-sync` for the full SDK regeneration workflow.
The generated client code lives in `frontend/src/client/` — never hand-edit it.

## Architecture

### Backend: Dependency Injection Chain

The most important design pattern. FastAPI's `Depends()` creates a call chain:

```
TokenDep (JWT from header)
    ↓
get_current_user(session, token) → decodes JWT, queries DB, returns User
    ↓
CurrentUser (Annotated alias)
    ↓
get_current_active_superuser(current_user) → checks is_superuser flag
```

Three `Annotated` type aliases in [backend/app/api/deps.py](backend/app/api/deps.py) make route signatures concise:

| Alias | Type | Source |
|-------|------|--------|
| `SessionDep` | `Session` | `get_db()` — one DB session per request, auto-closed |
| `TokenDep` | `str` | `OAuth2PasswordBearer` — extracts Bearer token |
| `CurrentUser` | `User` | `get_current_user()` — full User ORM object |

Route functions declare what they need in parameters and FastAPI injects them. `*` as first parameter forces keyword-only arguments after it.

### Backend: SQLModel Dual-Purpose Classes

In [backend/app/models.py](backend/app/models.py), the same class can be both a DB table and a Pydantic validation schema. The `table=True` parameter is the switch:

- **With `table=True`**: `User`, `Item` — maps to a database table via SQLAlchemy
- **Without `table=True`**: `UserCreate`, `UserPublic`, `ItemUpdate`, `Token`, `Message` — pure Pydantic schemas for request/response validation

Inheritance pattern: `UserBase` (shared fields) → `User(UserBase, table=True)` + `UserCreate(UserBase)` + `UserPublic(UserBase)`. Creates, updates, and public responses each get their own variant with different required/optional fields.

### Backend: Permission Patterns

Two patterns coexist — see [api-conventions.md](.claude/rules/api-conventions.md) for full code examples:

1. **Route-level guard** — `dependencies=[Depends(get_current_active_superuser)]` on the decorator. Every access requires admin.
2. **Inline check** — `if not current_user.is_superuser:` inside the function body. Behavior differs by role.

Don't mix both — pick one per endpoint.

### Backend: Password Security

[backend/app/core/security.py](backend/app/core/security.py) uses `pwdlib` with two hashers:
- **Argon2** (primary) — modern, memory-hard
- **Bcrypt** (fallback) — for existing passwords from older deployments

`verify_and_update()` returns `(verified: bool, new_hash: str | None)`. If a password was stored as Bcrypt, verifying it returns the Argon2 re-hash — the `authenticate()` function in [backend/app/crud.py](backend/app/crud.py) automatically persists the upgrade.

**Timing attack defense**: When a login email doesn't exist, `authenticate()` still runs `verify_password()` against a `DUMMY_HASH` constant — an Argon2 hash whose verification time matches a real user lookup, preventing email enumeration via response timing.

### Frontend: Auto-Generated Type-Safe API Client

The pipeline: FastAPI auto-generates OpenAPI schema at `/api/v1/openapi.json` → [scripts/generate-client.sh](scripts/generate-client.sh) downloads it → `@hey-api/openapi-ts` ([frontend/openapi-ts.config.ts](frontend/openapi-ts.config.ts)) generates TypeScript SDK into `frontend/src/client/`.

Usage in components:
```ts
import { UsersService } from "@/client"
const user = await UsersService.readUserById({ path: { user_id: id } })
```

Global config in [frontend/src/main.tsx](frontend/src/main.tsx): `OpenAPI.BASE` from `VITE_API_URL`, `OpenAPI.TOKEN` reads `localStorage("access_token")`. 401/403 errors automatically clear the token and redirect to `/login`.

### Frontend: Route Architecture

TanStack Router with file-based routing. Key files:

| File | Role |
|------|------|
| [frontend/src/routes/__root.tsx](frontend/src/routes/__root.tsx) | Root layout, providers |
| [frontend/src/routes/_layout.tsx](frontend/src/routes/_layout.tsx) | Authenticated layout (sidebar + content) |
| [frontend/src/routes/login.tsx](frontend/src/routes/login.tsx) | Public — `beforeLoad` redirects if already logged in |
| [frontend/src/routes/_layout/](frontend/src/routes/_layout/) | All authenticated pages |
| [frontend/src/routeTree.gen.ts](frontend/src/routeTree.gen.ts) | Auto-generated by `@tanstack/router-cli` |

### Docker Service Startup Order

```
db (healthy)
  ↓
prestart (runs alembic upgrade head + initial data seed)
  ↓
backend (FastAPI, health-checked at /api/v1/utils/health-check/)
frontend (Nginx serving static SPA, built with VITE_API_URL baked in)
```

Traefik is the reverse proxy. In production ([compose.traefik.yml](compose.traefik.yml)), it runs standalone on the `traefik-public` network and handles Let's Encrypt TLS. In development ([compose.override.yml](compose.override.yml)), a minimal Traefik is bundled for subdomain routing on `localhost.tiangolo.com`.

### Configuration Flow

`.env` at project root → `pydantic-settings` in [backend/app/core/config.py](backend/app/core/config.py) → `Settings` singleton. Key computed fields:
- `SQLALCHEMY_DATABASE_URI` — built from POSTGRES_* vars via `PostgresDsn.build()`
- `all_cors_origins` — BACKEND_CORS_ORIGINS + FRONTEND_HOST
- `emails_enabled` — true only when SMTP_HOST and EMAILS_FROM_EMAIL are both set

In production, the `_enforce_non_default_secrets` validator raises an error if SECRET_KEY/POSTGRES_PASSWORD/FIRST_SUPERUSER_PASSWORD is still `"changethis"`.

## Monorepo Structure

Root [package.json](package.json) declares `"workspaces": ["frontend"]`. Bun/npm hoists frontend dependencies to root `node_modules/`. Scripts at root delegate to frontend via `--filter frontend`.
