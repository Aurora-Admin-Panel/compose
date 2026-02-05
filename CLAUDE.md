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

Access new frontend at http://localhost:8001 (nginx2 → frontend via nginx_v2.conf).
Legacy frontend at http://localhost:8000 (nginx → frontend-old). Database admin at http://localhost:8070.

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
Three Git submodules: `backend/`, `frontend/`, `deploy/`. The root repo has `docker-compose.yml` as the orchestration entry point. `frontend-old/` is deprecated—do not modify.

### Request flow
```
Client :8001 → nginx2 → /api/*  → backend:8888 (FastAPI)
                       → /*     → frontend:5173 (Vite)
```
WebSocket connections (GraphQL subscriptions) pass through nginx with Upgrade headers.

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
- **GraphQL client:** Apollo Client configured in `src/grapgql.js` (typo in filename). Queries/mutations/subscriptions in `src/quries/` (also typo).
- **Auth:** JWT token stored in localStorage via Jotai `atomWithStorage('auth')`. Apollo auth middleware injects Bearer token. `useAuthReducer()` hook manages login/logout.
- **Routing:** React Router 6 in `App.jsx`. Protected routes under `/app/*` via `Layout.jsx`.
- **i18n:** i18next with English and Chinese. Translations in `public/locales/`.
- **Modals:** Centralized `ModalManager` in `src/features/modal/` routes by modal type.

### Database

PostgreSQL with TimescaleDB extension for time-series metric data. Credentials in docker-compose.yml: `aurora/AuroraAdminPanel321/aurora`. Async connections use `asyncpg`, sync use `psycopg2`.

### Task Queue

Huey with Redis backend. The `worker` service runs the same Docker image as `backend` but executes Huey consumer. Tasks manage SSH connections to relay servers, collect metrics, sync traffic, and execute forwarding rule changes.

## Git Conventions

- Do not add `Co-Authored-By` lines to commits. Commit as the user.

## Key Conventions

- GraphQL permission classes are applied per-field: `permission_classes=[IsAuthenticated]`
- Port forwarding methods are defined in `MethodEnum` (14+ methods). Adding a new method requires: DB enum migration → task implementation in `tasks/functions/` → orchestrator registration → frontend UI.
- Server config and port config use JSON columns (`MutableDict`) for flexible key-value storage.
- Real-time data flows: backend collects metrics on a schedule → publishes to Redis pub/sub → GraphQL subscription resolvers yield to connected clients.
- File uploads go through GraphQL mutation with `apollo-upload-client` on frontend, stored at `FILE_STORAGE_PATH` (`/app/files`).

## Known Issues

- `frontend/src/grapgql.js` filename typo (should be `graphql`)
- `frontend/src/quries/` folder name typo (should be `queries`)
- `SECREY_KEY` env var in docker-compose.yml is a typo (should be `SECRET_KEY`)
- Root README.md is outdated (references React 16, Python 3.8)
