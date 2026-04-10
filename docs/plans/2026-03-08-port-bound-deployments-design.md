# Port-Bound Deployments

## Problem

Deployments and ports are separate systems with no link between them. In a multi-tenant setup, users are allocated ports via `PortUser`, but when deploying a service there's no way to tie it to a specific port. The service author can't reference the allocated port number in command args, env vars, or config files.

## Decision

Use Approach A: `requiresPort` toggle on the service definition + `port_id` FK on `ServerDeployment` + `{{port}}` context variable injection during compilation.

## Design

### Data Model

**ServerDeployment** — one new column:
- `port_id` (Integer, FK -> port.id, nullable) — null for services that don't require a port
- Partial unique constraint: `(port_id) WHERE is_active = True AND port_id IS NOT NULL` — one active deployment per port

**Service definition schema** (`aurora-exec/v1`) — one new top-level field:
- `requiresPort` (boolean, default `true`) — when true, deployment must include a port selection

No changes to Port, PortUser, ServiceBinding, or DeploymentLog.

### Compilation & Context Injection

`compile_service_preview()` changes:
- `context_payload` gains optional `port` key (integer)
- When `requiresPort` is true and `context.port` is present, `{{port}}` is available as a template variable in: param defaults, `baseArgs`, file `pathTemplate`, file content, and any string in the emit pipeline
- If `requiresPort` is true but `context.port` is missing, return `ok: false` with error
- Preview mode (service editor): pass `context.port = 0` as placeholder

### GraphQL API

**Mutations** — `deployExecutable` and `deployService` gain optional `portId: Int`:
- If `requiresPort: true`, `portId` is required
- If `requiresPort: false`, `portId` must be null
- Validates: port belongs to target server, user has access, port has no active deployment
- Sets `port_id` on ServerDeployment, passes `Port.num` as `context.port` to compilation

**Queries:**
- `ServerDeploymentType` gains `port` field
- New query: `availablePortsForDeployment(serverId: Int!) -> [Port!]!`
  - Superuser: all ports on any server with no active deployment
  - Admin (is_ops on server): all ports on that server with no active deployment
  - Normal user: only ports allocated via PortUser with no active deployment

### Frontend

**DeployModal:**
- If `requiresPort` is true, show port selector dropdown before param form
- Fetch via `availablePortsForDeployment(serverId)`
- Display as "Port {num}" or "Port {num} -> {external_num}" if external differs
- No available ports = disable deploy with message
- Pass selected port's `num` as `context.port` to auto-compile preview
- Pass `portId` to deploy mutation

**Deployment list/card UI:**
- Show port number alongside deployment info when present

**Service editor:**
- No changes — `{{port}}` is a context variable authors type manually
