# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Summary

Aurora is a multi-server port forwarding management panel. Administrators add remote servers via SSH, allocate ports to users, and configure forwarding rules (iptables, gost, realm, v2ray, brook, etc.). The panel monitors server metrics in real-time and tracks per-user traffic. It runs as a Docker Compose stack on a control server and manages relay servers via SSH/Ansible.

## Development Commands

### Starting the environment
```bash
docker-compose up -d
docker-compose exec backend alembic upgrade head    # run migrations
docker-compose exec backend python3 app/initial_data.py  # seed initial superuser
```

Access frontend at http://aurora.localhost:8080 (nginx-proxy → frontend).
API at http://aurora.localhost:8080/api (nginx-proxy → backend). Database admin at http://db.localhost:8080.

### Backend
```bash
docker-compose build backend                   # rebuild after dependency changes
docker-compose restart backend worker          # restart after code changes (DEV mode has hot-reload)
docker-compose logs -f backend                 # view backend logs
docker-compose logs -f worker                  # view worker/task logs
docker-compose exec backend pytest             # run all backend tests
docker-compose exec backend pytest app/tests/test_specific.py  # single test
docker-compose exec backend alembic revision --autogenerate -m "description"  # new migration
```

### Frontend
```bash
cd frontend && npm install && npm run dev      # local dev server on :5173
cd frontend && npm run build                   # production build
```

### Full test suite
```bash
./scripts/test.sh
```

## Architecture

### Monorepo with submodules
Three Git submodules: `backend/`, `frontend/`, `deploy/`. The root repo has `docker-compose.yml` as the orchestration entry point.

### Request flow
```
Client :8080 → nginx-proxy → /api/*  → backend:8888 (FastAPI)   [VIRTUAL_HOST=aurora.localhost]
                            → /*     → frontend:5173 (Vite)     [VIRTUAL_HOST=aurora.localhost]
```
WebSocket connections (GraphQL subscriptions) pass through nginx-proxy with Upgrade headers.

### Backend (`backend/`)

**FastAPI + Strawberry GraphQL** is the primary API at `/api/graphql`. REST endpoints in `app/api/` are deprecated except `POST /api/token` for OAuth2 login.

Key directories:
- `app/graphql/` — Strawberry types, queries, mutations, subscriptions. This is where API changes happen.
- `app/graphql/permission.py` — Permission classes: `IsAuthenticated`, `IsAdmin` (is_ops OR is_superuser), `IsSuperUser`
- `app/db/models/` — SQLAlchemy 1.4 models. Core: User, Server, ServerUser (M2M), Port, PortUser (M2M), PortForwardRule, File. Time-series: ServerMetric, DiskUsage, NetworkCounter (TimescaleDB).
- `app/db/async_session.py` — Async session factory (asyncpg). Use `async_db_session()` context manager.
- `app/core/config.py` — All settings read from environment variables.
- `app/core/redis_client.py` — Async Redis connection pools (separate pool for pub/sub).
- `app/alembic/versions/` — 21 migration files.
- `tasks/` — Huey task queue. `tasks/server.py` has main tasks (connect_runner2, servers_usage_runner). `tasks/functions/` has forwarding method implementations. `tasks/utils/` has SSH connection management, metric collection, traffic tracking.
- `ansible/` — Ansible playbooks for relay server provisioning, deprecated.

GraphQL subscriptions use Redis Streams (`task_stream`) and Redis pub/sub (`server_metric`) for real-time data.

### Frontend (`frontend/`)

**React 18 + Vite 7 + Tailwind CSS 4 + DaisyUI 5**. Feature-based organization under `src/features/`.

Key patterns:
- **State management:** Jotai atoms in `src/atoms/` are primary (auth, theme, notifications, modals). Redux store in `src/store/` is legacy.
- **GraphQL client:** Apollo Client configured in `src/graphql.js`. Queries/mutations/subscriptions in `src/queries/`.
- **Auth:** JWT token stored in localStorage via Jotai `atomWithStorage('auth')`. Apollo auth middleware injects Bearer token. `useAuthReducer()` hook manages login/logout.
- **Routing:** React Router 6 in `App.jsx`. Protected routes under `/app/*` via `Layout.jsx`.
- **i18n:** i18next with English and Chinese. Translations in `public/locales/`.
- **Modals:** Centralized `ModalManager` in `src/features/modal/` routes by modal type.
- **Hooks:** Generic, reusable hooks belong in `src/hooks/`. Feature-specific hooks stay in their feature directory.

### Database

PostgreSQL with TimescaleDB extension for time-series metric data. Credentials in docker-compose.yml: `aurora/AuroraAdminPanel321/aurora`. Async connections use `asyncpg`, sync use `psycopg2`.

### Task Queue

Huey with Redis backend. The `worker` service runs the same Docker image as `backend` but executes Huey consumer. Tasks manage SSH connections to relay servers, collect metrics, sync traffic, and execute forwarding rule changes.

## Git Conventions

- Do not add `Co-Authored-By` lines to commits. Commit as the user.

## Change Policy

This is a single-owner personal project. Backward compatibility does not need to be preserved — large refactors, rewrites, and API changes are all fine. The only constraint is that database migrations must not be destructive (no dropping columns/tables that contain data needed by the running system). Everything else (frontend, backend APIs, config, tooling) can be changed freely.

## Key Conventions

- GraphQL permission classes are applied per-field: `permission_classes=[IsAuthenticated]`
- Port forwarding methods are defined in `MethodEnum` (14+ methods). Adding a new method requires: DB enum migration → task implementation in `tasks/functions/` → orchestrator registration → frontend UI.
- Server config and port config use JSON columns (`MutableDict`) for flexible key-value storage.
- Real-time data flows: backend collects metrics on a schedule → publishes to Redis pub/sub → GraphQL subscription resolvers yield to connected clients.
- File uploads go through GraphQL mutation with `apollo-upload-client` on frontend, stored at `FILE_STORAGE_PATH` (`/app/files`).

## Service Definition System

Service definitions define a parameterised command schema (`aurora-exec/v1`) that can be compiled into a concrete shell invocation with args, env vars, files, and stdin.

### Service schema (authoring format)
The JSON service definition is validated by Pydantic model `ServiceDefinitionAuthoringV1` in `backend/app/db/schemas/service_definition.py`. Key shapes:
- **Top-level**: `schemaVersion`, `contractKey`, `version`, `title`, `description`, `exec`, `ui`, `params[]`
- **`exec`**: `bin`, `baseArgs[]`, `workingDir`, `timeoutSeconds`, `source` (optional, for binary acquisition)
- **`params[]`**: each has `key`, `type` (string/int/float/bool/enum/secret/list/object), `label`, `required`, `default`, `emit`, `validation`, `conditions`, `ui`, `secret`
- **`emit`**: exactly one target — `arg`, `flag`, `flagTrue`/`flagFalse`, `env`, `pos`, `file` (pathTemplate+format), `stdin` (format). Also `mode` (repeat/csv for lists), `emitIf`, `separator`

### Backend compilation
`backend/app/utils/service_definition.py` — `compile_service_preview()` takes a service definition dict + user-submitted values + context → returns `{ok, plan, preview, warnings}`. Pipeline: validate via Pydantic → coerce/validate each param value → evaluate conditions → emit each param into argv/env/files/stdin → build shell preview with secret redaction.

GraphQL entry points in `backend/app/graphql/service_definition.py`:
- `compileServicePreview(contract, values, context)` — compile from raw JSON
- `compileServicePreviewById(id, values, context)` — compile from saved service definition
- CRUD mutations: `createServiceDefinition`, `updateServiceDefinition`, `deleteServiceDefinition`

### Frontend rendering
The service editor UI lives in `frontend/src/features/service-editor/`:
- **`serviceAdapter.js`** — `serviceDefinitionToDynamicSchema(contract)` converts the authoring param array into a flat `{key: fieldSpec}` object that the form renderer understands. Maps param types (string→text, int/float→number, bool→checkbox, enum→select, secret→password, list, object) and attaches validation/grid config.
- **`useDynamicForm.jsx`** — hook that takes the adapted schema → creates a `react-hook-form` instance → renders `FieldsRenderer` inside a grid container. Returns `{form, methods}`. Supports `onValuesChange` callback for live auto-compile.
- **`fields/FieldsRenderer.jsx`** — recursively renders field components (TextField, SelectField, CheckboxField, ListField, ObjectField) based on schema type. Supports nested objects and arrays with parent path tracking.
- **`ParamEditorPanel.jsx`** — visual editor for authoring service params (key, type, emit preset, validation, etc.). Mutates the service definition JSON draft in-place.
- **`ServiceEditorPage.jsx`** — orchestrates the three panels (AuthoringJsonPanel, FormPreviewPanel, CompileOutputPanel) + ParamEditorPanel. Manages auto-compile with debounce.

The `DeployModal` (`features/deployment/DeployModal.jsx`) also uses `useDynamicForm` + `serviceDefinitionToDynamicSchema` to render parameter forms when deploying a service to a server.

## Design Context

### Users
Mixed audience: personal sysadmins managing a few relay servers, small ops teams running infrastructure for a group, and service providers with many end users. The interface must scale from simple single-server setups to complex multi-server deployments without overwhelming either audience.

### Brand Personality
**Modern, sleek, confident.** Aurora should feel like a polished professional tool — not a hobbyist dashboard, not enterprise bloatware. It communicates competence and control.

### Aesthetic Direction
**Minimal & functional** — clean lines, generous whitespace, content-first layouts. Reference points: Linear, Vercel. Avoid visual clutter, decorative elements, and unnecessary chrome. Every element should earn its place. Data density is acceptable when purposeful (metrics, server status), but default to breathing room.

Anti-references: Grafana-style visual overload, generic Bootstrap admin templates, overly playful/cartoon UIs.

### Design Principles
1. **Content over decoration** — Let data and controls speak. No ornamental gradients, shadows, or borders that don't serve hierarchy.
2. **Mobile-first, desktop-refined** — The mobile experience is a first-class citizen, not a responsive afterthought. Touch targets, drawer navigation, and card layouts should work naturally on small screens.
3. **Consistent semantic color** — Use DaisyUI's semantic palette (primary, success, warning, error) consistently. Purple is the brand accent. Color always communicates meaning.
4. **Progressive disclosure** — Show essential information first; reveal complexity on demand. Dropdowns, expandable sections, and modals over cramming everything on screen.
5. **Quiet confidence** — Subtle transitions, restrained animations (Framer Motion), no flashy effects. The interface should feel fast, stable, and trustworthy.

### Design System
- **Framework**: Tailwind CSS 4 + DaisyUI 5 (component primitives)
- **Icons**: Lucide React (primary)
- **Animation**: Framer Motion (subtle, purposeful only)
- **Charts**: Recharts
- **Themes**: 3 custom (Aurora Classic light, Sunset dark, Morning light) + DaisyUI built-ins
- **Typography**: System font stack (no custom fonts)
- **Border radius**: 1rem containers, 0.5rem selectors, 0.25rem small elements
- **i18n**: English + Chinese (zh fallback)

## Known Issues

- Root README.md is outdated (references React 16, Python 3.8)
