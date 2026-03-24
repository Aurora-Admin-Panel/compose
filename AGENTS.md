# AGENTS.md - Aurora Admin Panel

## Project Overview

Aurora is a multi-server port forwarding and relay management panel. Administrators use it to:

- Add and manage remote servers over SSH
- Allocate ports and deployments to users
- Configure forwarding rules across multiple relay methods
- Monitor server and traffic metrics in real time
- Manage uploaded files and generated configs
- Define reusable service definitions and deploy them to servers

The control plane runs as a local Docker Compose stack and manages remote relay servers through backend tasks, SSH, and optional Ansible helpers.

## Repository Structure

This repository is a monorepo with four Git submodules:

```text
aurora/
├── backend/            # FastAPI + Strawberry GraphQL backend (submodule)
├── frontend/           # Active React/Vite admin frontend (submodule)
├── frontend-old/       # Deprecated legacy frontend (submodule, do not modify)
├── deploy/             # Deployment configs and install scripts (submodule)
├── docker-compose.yml  # Development orchestration entry point
├── AGENTS.md           # Canonical repo guidance
└── CLAUDE.md           # Symlink to AGENTS.md
```

## Architecture

```text
Client :8080 → nginx-proxy → /api/* → backend:8888
                          → /*     → frontend:5173

db.localhost:8080 → nginx-proxy → adminer:8080

backend → PostgreSQL + TimescaleDB
        → Redis

worker → Redis job queue
       → PostgreSQL
       → SSH / orchestration on managed servers
```

GraphQL subscriptions are served from `/api/graphql` over WebSocket and pass through `nginx-proxy`.

### Key Services

| Service | Port | Purpose |
|---|---:|---|
| `nginx-proxy` | `8080` | Reverse proxy for `aurora.localhost` and `db.localhost` |
| `backend` | `8888` | FastAPI app, GraphQL API, auth, uploads |
| `worker` | - | Huey worker for background jobs and SSH orchestration |
| `frontend` | `5173` | Vite dev server for the active frontend |
| `postgres` | `5432` | PostgreSQL + TimescaleDB |
| `redis` | `6379` | Cache, queue, pub/sub |
| `adminer` | proxied via `db.localhost:8080` | Database admin UI |

Note: local developer environments sometimes remap the proxy to a different host port such as `8060`, but the committed `docker-compose.yml` exposes `8080`.

## Backend (`backend/`)

### Tech Stack

- FastAPI
- Strawberry GraphQL
- SQLAlchemy 1.4 with async `asyncpg`
- Alembic
- Huey
- Redis
- Fabric / Paramiko

### Important Backend Areas

```text
backend/app/
├── main.py                    # FastAPI app setup and route mounting
├── initial_data.py            # Seed initial superuser
├── core/                      # Settings, auth, security, Redis clients
├── graphql/                   # Primary API surface
├── db/
│   ├── models/                # SQLAlchemy models
│   ├── crud/                  # Data access helpers
│   ├── schemas/               # Pydantic schema definitions
│   ├── session.py             # Sync session
│   └── async_session.py       # Async session
├── api/                       # Legacy REST endpoints
├── alembic/                   # Migrations
└── utils/                     # Compilation, permissions, file helpers

backend/tasks/
├── server.py                  # Main task entrypoints
├── functions/                 # Forwarding-method implementations
└── utils/                     # SSH, orchestration, metrics, traffic
```

### GraphQL API

- Endpoint: `/api/graphql`
- Transport: HTTP queries/mutations + WebSocket subscriptions
- Auth: JWT bearer token, with `POST /api/token` still used for login
- Permission classes live in `backend/app/graphql/permission.py`

Current GraphQL modules include:

- `auth.py`
- `deployment.py`
- `file.py`
- `metric.py`
- `port.py`
- `port_forward.py`
- `server.py`
- `service_definition.py`
- `task.py`
- `user.py`

### Core Models

Core data models include:

- `User`
- `Server`
- `Port`
- `PortForwardRule`
- `File`
- `ServerDeployment`
- `ServiceDefinition`
- `ServiceBinding`
- time-series metric models in `metric.py`

### Common Backend Commands

```bash
docker compose up -d
docker compose logs -f backend
docker compose logs -f worker
docker compose exec backend alembic upgrade heads
docker compose exec backend python app/initial_data.py
docker compose exec backend pytest
docker compose exec backend alembic revision --autogenerate -m "description"
```

## Frontend (`frontend/`)

### Tech Stack

- React 19
- TypeScript
- Vite 7
- Tailwind CSS 4
- shadcn/ui component layer on top of Radix primitives
- Apollo Client 3 with `graphql-ws` and `apollo-upload-client`
- Jotai
- React Router 6
- React Hook Form
- i18next
- Framer Motion
- Recharts

### Frontend Structure

```text
frontend/src/
├── main.tsx                   # Root providers and app mount
├── App.tsx                    # Route tree
├── Layout.tsx                 # Authenticated shell
├── routes.ts                  # Canonical route config
├── graphql.ts                 # Apollo client setup
├── index.css                  # Tailwind v4 theme tokens and globals
├── components/
│   ├── theme-provider.tsx
│   └── ui/                    # shadcn/Radix-based primitives
├── atoms/                     # Jotai atoms and reducers
├── features/
│   ├── auth/
│   ├── server/
│   ├── port/
│   ├── user/
│   ├── file/
│   ├── deployment/
│   ├── service-editor/
│   ├── layout/
│   ├── modal/
│   ├── theme/
│   ├── i18n/
│   └── ui/
├── hooks/
├── queries/                   # GraphQL operations
├── types/generated.ts         # Generated GraphQL types
└── utils/
```

### Frontend Runtime Patterns

- Root provider stack in `main.tsx` is:
  `ApolloProvider` → `ThemeProvider` → `TooltipProvider` → `HelmetProvider` → `Suspense`
- shadcn/Radix primitives live under `src/components/ui/`
- Global app state is handled with Jotai atoms in `src/atoms/`
- Route metadata lives in `src/routes.ts`
- GraphQL operations live in `src/queries/`
- The active app is TypeScript-first; new work should follow the TS/shadcn structure, not the old JSX/DaisyUI layout

### Current Routes

```text
/login
/create-account
/app
  /app/servers
  /app/servers/:serverId
  /app/servers/:serverId/ports
  /app/servers/:serverId/users
  /app/users
  /app/files
  /app/services
  /app/services/editor
  /app/services/editor/:serviceId
  /app/about
  /app/themes
```

`/app/deployments` currently redirects back to the server area rather than owning a separate page.

### Auth Flow

1. `POST /api/token` with email/password
2. JWT is stored in localStorage via Jotai `authAtom`
3. `frontend/src/graphql.ts` injects the bearer token for GraphQL requests
4. GraphQL subscriptions connect to `ws://<host>/api/graphql`
5. Permission state is derived from the decoded JWT payload

### Common Frontend Commands

```bash
docker compose logs -f frontend
cd frontend && npm run dev
cd frontend && npm run build
```

Access the active app through `http://aurora.localhost:8080`.

## Service Definition System

Aurora now uses service-definition terminology in the UI and database model layer.

### Backend

- Schema model: `backend/app/db/schemas/service_definition.py`
- DB model: `backend/app/db/models/service_definition.py`
- GraphQL API: `backend/app/graphql/service_definition.py`
- Compiler: `backend/app/utils/service_definition.py`

`compile_service_preview()` validates authoring JSON, coerces submitted values, applies conditions, emits args/env/files/stdin, and returns a preview plan.

### Frontend

The service editor lives in `frontend/src/features/service-editor/` and includes:

- `ServiceListPage.tsx`
- `ServiceEditorPage.tsx`
- `ParamEditorPanel.tsx`
- `serviceAdapter.ts`
- `useDynamicForm.tsx`
- preview panels for authoring JSON, compiled output, and rendered forms

The deployment UI also consumes the same service-definition schema to render parameter forms dynamically.

### Naming Quirk

The rename from contracts to services is not fully complete at the schema boundary yet:

- authoring JSON still uses `contractKey`
- some compiler/template variables still reference `contractKey`
- `compileServicePreview` still takes a `contract` JSON argument

When updating this area, prefer service terminology in new UI and model code, but expect those schema names to remain until a dedicated cleanup pass lands.

## Development Workflow

### Starting the Stack

```bash
docker compose up -d
docker compose exec backend alembic upgrade heads
docker compose exec backend python app/initial_data.py
```

Then open:

- `http://aurora.localhost:8080`
- `http://aurora.localhost:8080/api/graphql`
- `http://db.localhost:8080`

### Typical Change Areas

- GraphQL API changes: `backend/app/graphql/`
- DB model changes: `backend/app/db/models/` plus Alembic
- Task / SSH orchestration: `backend/tasks/`
- UI components and pages: `frontend/src/features/` and `frontend/src/components/ui/`
- Route and navigation changes: `frontend/src/routes.ts`, `frontend/src/Layout.tsx`, `frontend/src/features/layout/`

## Change Policy

This is a single-owner project. Large refactors and internal API changes are acceptable. Backward compatibility is not a hard requirement, except for preserving user data and avoiding destructive database migrations unless explicitly intended.

## Known Quirks

- `frontend-old/` is deprecated and should not be modified unless explicitly requested.
- REST is still used for login and some legacy flows; GraphQL is the primary application API.
- The service-definition schema still carries `contractKey` naming internally.
- `frontend/src/utils/websocketManager.ts` exists as leftover utility code, but the active app uses GraphQL subscriptions.
- Fresh `npm install` can currently hit an Apollo peer-dependency conflict with React 19; `--legacy-peer-deps` is only a temporary local workaround, not the intended final fix.
