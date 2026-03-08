# Port-Bound Deployments Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Link deployments to allocated ports so services can bind to user-assigned ports via `{{port}}` context variable injection.

**Architecture:** Add `port_id` FK (nullable) to `ServerDeployment`, `requiresPort` boolean to the service definition schema, and `{{port}}` template variable substitution in `compile_service_preview()`. Frontend shows a port selector when deploying port-requiring services.

**Tech Stack:** SQLAlchemy 1.4, Alembic, Pydantic, Strawberry GraphQL, React 18, Apollo Client

---

### Task 1: Alembic Migration — add `port_id` to `server_deployment`

**Files:**
- Create: `backend/app/alembic/versions/<generated>_add_port_id_to_server_deployment.py`

**Step 1: Generate migration**

Run: `docker-compose exec backend alembic revision --autogenerate -m "add port_id to server_deployment"`

Then edit the generated migration to contain:

```python
from alembic import op
import sqlalchemy as sa

revision = "<generated>"
down_revision = "38fd48957fee"
branch_labels = None
depends_on = None


def upgrade():
    op.add_column(
        "server_deployment",
        sa.Column("port_id", sa.Integer(), sa.ForeignKey("port.id"), nullable=True),
    )
    # Partial unique index: only one active deployment per port
    op.create_index(
        "ix_server_deployment_port_id_active",
        "server_deployment",
        ["port_id"],
        unique=True,
        postgresql_where=sa.text("is_active = true AND port_id IS NOT NULL"),
    )


def downgrade():
    op.drop_index("ix_server_deployment_port_id_active", table_name="server_deployment")
    op.drop_column("server_deployment", "port_id")
```

**Step 2: Run migration**

Run: `docker-compose exec backend alembic upgrade head`
Expected: Migration applies successfully.

**Step 3: Commit**

```bash
git add backend/app/alembic/versions/*add_port_id*
git commit -m "migration: add port_id FK to server_deployment with partial unique index"
```

---

### Task 2: Update ServerDeployment model — add `port_id` column and relationship

**Files:**
- Modify: `backend/app/db/models/server_deployment.py`

**Step 1: Add column and relationship**

In `ServerDeployment` class (after `server_id` column, line 56), add:

```python
port_id = Column(Integer, ForeignKey("port.id"), nullable=True)
```

After the `server` relationship (line 76), add:

```python
port = relationship("Port", backref="deployment")
```

Using `backref` so we don't need to modify `port.py`. The backref gives `Port.deployment` (singular — one active deployment per port).

**Step 2: Verify import**

No new imports needed — `Integer`, `ForeignKey`, `Column`, `relationship` are already imported.

**Step 3: Commit**

```bash
git add backend/app/db/models/server_deployment.py
git commit -m "feat: add port_id column and relationship to ServerDeployment"
```

---

### Task 3: Add `requiresPort` to service definition schema

**Files:**
- Modify: `backend/app/db/schemas/service_definition.py`

**Step 1: Add field to ServiceDefinitionAuthoringV1**

At line 297 (after `params`), add:

```python
requiresPort: bool = True
```

No validators needed — it's a simple boolean with a default.

**Step 2: Add `port` to ALLOWED_TEMPLATE_VARS**

At line 10, change:

```python
ALLOWED_TEMPLATE_VARS = {"jobId", "contractKey", "paramKey"}
```

to:

```python
ALLOWED_TEMPLATE_VARS = {"jobId", "contractKey", "paramKey", "port"}
```

**Step 3: Commit**

```bash
git add backend/app/db/schemas/service_definition.py
git commit -m "feat: add requiresPort field to service definition schema"
```

---

### Task 4: Inject `{{port}}` in `compile_service_preview()`

**Files:**
- Modify: `backend/app/utils/service_definition.py`

**Step 1: Add context variable substitution**

The `{{port}}` variable needs to work in three places:
1. `baseArgs` — e.g., `["-L", "0.0.0.0:{{port}}"]`
2. Param `default` values — e.g., `"default": "0.0.0.0:{{port}}"`
3. File content and path templates (already uses `_render_path_template`)

Add a helper function after `_render_path_template` (after line 202):

```python
def _substitute_context_vars(value: t.Any, context: dict) -> t.Any:
    """Recursively substitute {{var}} placeholders in strings using context dict."""
    if isinstance(value, str):
        return PATH_TEMPLATE_VAR_RE.sub(
            lambda m: str(context.get(m.group(1), m.group(0))), value
        )
    if isinstance(value, list):
        return [_substitute_context_vars(item, context) for item in value]
    if isinstance(value, dict):
        return {k: _substitute_context_vars(v, context) for k, v in value.items()}
    return value
```

**Step 2: Apply substitution in `compile_service_preview()`**

After the schema validation succeeds (after line 384), add:

```python
# Validate requiresPort vs context
if contract.requiresPort:
    if not context_payload or "port" not in context_payload:
        return {"ok": False, "error": "This service requires a port selection"}

# Substitute context variables in baseArgs and param defaults
ctx = context_payload or {}
if contract.exec.baseArgs:
    contract.exec.baseArgs = [
        _substitute_context_vars(a, ctx) for a in contract.exec.baseArgs
    ]
for param in contract.params:
    if param.default is not None:
        param.default = _substitute_context_vars(param.default, ctx)
```

**Step 3: Update `_normalize_path_context` to include port**

At line 189, update the function to pass through the port:

```python
def _normalize_path_context(contract: ServiceDefinitionAuthoringV1, context: dict, param_key: str):
    ctx = {
        "jobId": str((context or {}).get("jobId", "preview")),
        "contractKey": contract.contractKey,
        "paramKey": param_key,
    }
    if "port" in (context or {}):
        ctx["port"] = str(context["port"])
    return ctx
```

**Step 4: Commit**

```bash
git add backend/app/utils/service_definition.py
git commit -m "feat: inject {{port}} context variable in service compilation"
```

---

### Task 5: Add `availablePortsForDeployment` GraphQL query

**Files:**
- Modify: `backend/app/graphql/port.py`
- Modify: `backend/app/graphql/schema.py`

**Step 1: Add resolver to Port class in `port.py`**

Add this static method to the `Port` class (after `get_port_count`, around line 185):

```python
@staticmethod
async def get_available_ports_for_deployment(
    info: Info,
    server_id: int,
) -> List["Port"]:
    """Return ports on this server that the current user can access
    and that have no active deployment."""
    from app.db.models import ServerDeployment as DBServerDeployment

    user = info.context["request"].state.user

    # Subquery: port_ids with an active deployment
    active_deployment_port_ids = (
        select(DBServerDeployment.port_id)
        .where(
            DBServerDeployment.is_active == True,
            DBServerDeployment.port_id.isnot(None),
        )
        .scalar_subquery()
    )

    stmt = select(DBPort).where(
        DBPort.server_id == server_id,
        DBPort.is_active == True,
        DBPort.id.notin_(active_deployment_port_ids),
    ).order_by(DBPort.num)

    # Access control: superuser sees all, ops sees server ports, others need PortUser
    if not user.is_superuser:
        if user.is_ops:
            # Check user is admin for this server
            admin_server_ids = select(DBServerUser.server_id).where(
                DBServerUser.user_id == user.id
            )
            stmt = stmt.where(
                or_(
                    DBPort.server_id.in_(admin_server_ids),
                    DBPort.id.in_(
                        select(DBPortUser.port_id).where(DBPortUser.user_id == user.id)
                    ),
                )
            )
        else:
            stmt = stmt.where(
                DBPort.id.in_(
                    select(DBPortUser.port_id).where(DBPortUser.user_id == user.id)
                )
            )

    async with async_db_session() as async_db:
        result = await async_db.execute(stmt)
        return result.scalars().unique().all()
```

**Step 2: Wire query in `schema.py`**

In the `Query` class (after `paginated_server_deployments`, around line 121), add:

```python
available_ports_for_deployment: List[Port] = strawberry.field(
    resolver=Port.get_available_ports_for_deployment,
    permission_classes=[IsAuthenticated],
)
```

Add `Port` to imports from `app.graphql.port` if not already there. Also ensure `List` from `typing` is imported.

**Step 3: Commit**

```bash
git add backend/app/graphql/port.py backend/app/graphql/schema.py
git commit -m "feat: add availablePortsForDeployment query"
```

---

### Task 6: Update deploy mutations to accept `portId`

**Files:**
- Modify: `backend/app/graphql/deployment.py`
- Modify: `backend/app/graphql/schema.py`

**Step 1: Update `deploy_executable_resolver`**

Add `port_id: Optional[int] = None` parameter (line 214). Add validation inside the `async with` block, before the server loop:

```python
# Validate port if provided
port = None
if port_id is not None:
    port = (await db.execute(
        select(DBPort).where(DBPort.id == port_id)
    )).scalars().first()
    if not port:
        raise ValueError(f"Port {port_id} not found")
    # Check port belongs to one of the target servers
    if port.server_id not in server_ids:
        raise ValueError(f"Port {port_id} does not belong to any of the target servers")
    # Check no active deployment on this port
    active_on_port = (await db.execute(
        select(DBServerDeployment).where(
            DBServerDeployment.port_id == port_id,
            DBServerDeployment.is_active == True,
        )
    )).scalars().first()
    if active_on_port:
        raise ValueError(f"Port {port_id} already has an active deployment")
```

Import `DBPort` at the top of the file:
```python
from app.db.models import Port as DBPort
```

Set `port_id` on the deployment object (both create and update paths):
```python
deployment.port_id = port_id  # None if no port required
```

Note: when `port_id` is set and there are multiple `server_ids`, the port can only belong to one server. The validation above ensures the port's `server_id` is in the list. In practice, port-bound deployments will typically target a single server.

**Step 2: Update `deploy_service_resolver`**

Same changes as step 1 — add `port_id: Optional[int] = None` parameter and the same validation block.

Additionally, after loading the service definition, validate `requiresPort`:

```python
import json
config = json.loads(service.config_json) if isinstance(service.config_json, str) else service.config_json
requires_port = config.get("requiresPort", True)
if requires_port and port_id is None:
    raise ValueError("This service requires a port selection")
if not requires_port and port_id is not None:
    raise ValueError("This service does not accept a port")
```

Do the same for `deploy_executable_resolver` by loading the service definition through the binding.

**Step 3: Pass `context.port` to compilation in task dispatch**

The compilation happens in the Huey task, not in the resolver. The `port_id` is stored on `ServerDeployment`, so the task can load the port. No changes needed in the resolver for compilation — this is handled in Task 7.

**Step 4: Update schema.py mutation wiring**

The strawberry field definitions in `schema.py` don't need changes because the resolver signature change is picked up automatically by Strawberry (it inspects the function signature).

**Step 5: Commit**

```bash
git add backend/app/graphql/deployment.py
git commit -m "feat: accept portId in deploy mutations with validation"
```

---

### Task 7: Pass `context.port` in Huey deployment tasks

**Files:**
- Modify: `backend/tasks/deployment.py`

**Step 1: Find where `compile_service_preview` is called**

In the deployment task, find where the service definition is compiled and add `context.port`:

```python
# After loading the deployment and its port:
context = {"jobId": str(deployment.id)}
if deployment.port_id and deployment.port:
    context["port"] = deployment.port.num
```

Pass this context to `compile_service_preview()`.

**Step 2: Ensure port relationship is loaded**

When loading the deployment in the task, eagerly load the port:

```python
stmt = select(DBServerDeployment).where(
    DBServerDeployment.id == deployment_id
).options(joinedload(DBServerDeployment.port))
```

**Step 3: Commit**

```bash
git add backend/tasks/deployment.py
git commit -m "feat: pass port number as context variable to service compilation"
```

---

### Task 8: Update ServerDeploymentType to expose port

**Files:**
- Modify: `backend/app/graphql/deployment.py`

**Step 1: Add port field to ServerDeploymentType**

In the `ServerDeployment` strawberry type, add:

```python
port_id: Optional[int]
port: Optional[Annotated["Port", strawberry.lazy("app.graphql.port")]]
```

Ensure the `set_options` method (if it exists) loads the port relationship when requested.

**Step 2: Commit**

```bash
git add backend/app/graphql/deployment.py
git commit -m "feat: expose port on ServerDeploymentType"
```

---

### Task 9: Frontend — add port query and update deploy mutations

**Files:**
- Modify: `frontend/src/queries/deployment.js`

**Step 1: Add available ports query**

```javascript
export const GET_AVAILABLE_PORTS = gql`
  query GetAvailablePorts($serverId: Int!) {
    availablePortsForDeployment(serverId: $serverId) {
      id
      num
      externalNum
    }
  }
`;
```

**Step 2: Update deploy mutations to include portId**

Update `DEPLOY_EXECUTABLE`:
```graphql
mutation DeployExecutable(
  $serviceBindingId: Int!
  $serverIds: [Int!]!
  $values: JSON!
  $portId: Int
) {
  deployExecutable(
    serviceBindingId: $serviceBindingId
    serverIds: $serverIds
    values: $values
    portId: $portId
  ) {
    id
    serviceBindingId
    serverId
    portId
    status
    createdAt
    updatedAt
  }
}
```

Update `DEPLOY_SERVICE` similarly with `$portId: Int` variable.

**Step 3: Add portId to deployment query responses**

Update `GET_SERVER_DEPLOYMENT` and `GET_PAGINATED_SERVER_DEPLOYMENTS` to include `portId` and `port { num externalNum }` fields.

**Step 4: Commit**

```bash
cd frontend && git add src/queries/deployment.js && git commit -m "feat: add port queries and portId to deploy mutations"
```

---

### Task 10: Frontend — add port selector to DeployModal

**Files:**
- Modify: `frontend/src/features/deployment/DeployModal.jsx`

**Step 1: Import and query available ports**

After the server ID is known (from `modalProps.serverId`), query available ports:

```javascript
import { useQuery } from "@apollo/client";
import { GET_AVAILABLE_PORTS } from "../../queries/deployment";

// Inside component:
const { data: portsData, loading: portsLoading } = useQuery(GET_AVAILABLE_PORTS, {
  variables: { serverId },
  skip: !serverId,
});
```

**Step 2: Add port selector state**

```javascript
const [selectedPortId, setSelectedPortId] = useState(null);
```

**Step 3: Render port selector**

When `requiresPort` is true on the loaded service definition (from `configJson`), render a select dropdown in Step 2 (before the param form):

```jsx
{requiresPort && (
  <div className="form-control w-full">
    <label className="label"><span className="label-text">Port</span></label>
    <select
      className="select select-bordered w-full"
      value={selectedPortId || ""}
      onChange={(e) => setSelectedPortId(Number(e.target.value) || null)}
    >
      <option value="">Select a port...</option>
      {portsData?.availablePortsForDeployment?.map((port) => (
        <option key={port.id} value={port.id}>
          Port {port.num}{port.externalNum && port.externalNum !== port.num ? ` → ${port.externalNum}` : ""}
        </option>
      ))}
    </select>
    {portsData?.availablePortsForDeployment?.length === 0 && (
      <p className="text-sm text-error mt-1">No available ports on this server</p>
    )}
  </div>
)}
```

**Step 4: Pass portId to deploy mutation**

Update the deploy call (around line 128) to include `portId: selectedPortId`:

```javascript
// Catalog mode
deployService({
  variables: {
    serviceId: selectedService.id,
    serverIds: [serverId],
    values: formValues,
    portId: selectedPortId,
  },
});

// Binding mode
deployExecutable({
  variables: {
    serviceBindingId: selectedBinding.id,
    serverIds: [serverId],
    values: formValues,
    portId: selectedPortId,
  },
});
```

**Step 5: Pass port to auto-compile context**

When calling `compileServicePreview` for live preview, include the port number:

```javascript
const selectedPort = portsData?.availablePortsForDeployment?.find(p => p.id === selectedPortId);
const context = {
  jobId: "preview",
  ...(selectedPort ? { port: selectedPort.num } : {}),
};
```

**Step 6: Disable deploy button when port required but not selected**

```javascript
const requiresPort = configJson?.requiresPort !== false; // default true
const canDeploy = !requiresPort || selectedPortId;
```

Disable the deploy button when `!canDeploy`.

**Step 7: Commit**

```bash
cd frontend && git add src/features/deployment/DeployModal.jsx && git commit -m "feat: add port selector to deploy modal"
```

---

### Task 11: Show port info on deployment list/cards

**Files:**
- Modify: relevant deployment list component in `frontend/src/features/deployment/`

**Step 1: Find the deployment list component**

Check `frontend/src/features/deployment/` for list or card components that render deployments.

**Step 2: Display port number**

Where deployment info is shown, add the port:

```jsx
{deployment.port && (
  <span className="badge badge-outline badge-sm">
    Port {deployment.port.num}
  </span>
)}
```

**Step 3: Commit**

```bash
cd frontend && git add -A && git commit -m "feat: display port number on deployment cards"
```

---

### Task 12: Update built-in service seeds

**Files:**
- Modify: `backend/app/seed_services.py` (if it exists)

**Step 1: Add `requiresPort` to existing built-in service definitions**

For services that need ports (proxies, forwarders): `requiresPort: true` (or omit, since it defaults to true).

For services that don't (monitoring agents like node_exporter, iperf): set `requiresPort: false`.

**Step 2: Update `{{port}}` usage in baseArgs or param defaults**

For port-requiring services, replace hardcoded port params with `{{port}}` references. For example, a gost service might have:

```json
"baseArgs": ["-L", "0.0.0.0:{{port}}"]
```

**Step 3: Commit**

```bash
git add backend/app/seed_services.py
git commit -m "feat: set requiresPort and use {{port}} in built-in service definitions"
```
