# Rename Executable Contracts to Services

**Date**: 2026-03-08
**Status**: Approved
**Scope**: Full-stack rename (DB, backend, GraphQL, frontend, i18n)

## Motivation

"Executable Contract" as a concept is confusing. The feature's core job is **service lifecycle management** — install, configure, start, stop, update, remove software on relay servers. The mental model should be a **Service Manager**, and the primary noun should be **Service**.

The underlying schema, compilation engine, and deployment machinery are sound. This is a rename and reframe, not a redesign.

## Terminology Mapping

| Current | New |
|---------|-----|
| Executable Contract | Service (Definition) |
| Contract Key | Service Key |
| Contract Builder | Service Editor |
| Contract List | Services |
| File Contract Binding | Service Binding |
| App Catalog | Service Catalog |
| Built-in Contract | Built-in Service |
| Deploy Contract | Deploy Service |

## Database Changes

### Table Renames

| Current | New |
|---------|-----|
| `executable_contract` | `service_definition` |
| `file_contract_binding` | `service_binding` |
| `server_deployment` | unchanged |
| `deployment_log` | unchanged |

### Column Renames

| Table | Current | New |
|-------|---------|-----|
| `service_definition` | `contract_key` | `service_key` |
| `service_definition` | `schema_json` | `config_json` |
| `service_binding` | `contract_id` | `service_id` |
| `server_deployment` | `contract_id` | `service_id` |
| `server_deployment` | `binding_id` | `service_binding_id` |

### Preserved

- `server_deployment` and `deployment_log` table names (correct for service instances)
- Internal JSON schema version `aurora-exec/v1` (changing it would break stored data)
- JSON field names inside stored payloads (`contractKey`, `exec`, `params`)
- Unique constraint becomes `unique(service_key, version)`

### Migration

Single Alembic migration using `ALTER TABLE ... RENAME TO` and `ALTER TABLE ... RENAME COLUMN`. Non-destructive, preserves all data.

## Backend Changes

### Models (`app/db/models/`)

| Current File | New File | Class |
|-------------|----------|-------|
| `executable_contract.py` | `service_definition.py` | `ServiceDefinition` |
| `file_contract_binding.py` | `service_binding.py` | `ServiceBinding` |
| `server_deployment.py` | unchanged | Update FK refs: `contract_id` -> `service_id`, `binding_id` -> `service_binding_id` |

### Schemas (`app/db/schemas/`)

- `executable_contract.py` -> `service_definition.py`
- `ExecutableContractAuthoringV1` -> `ServiceDefinitionAuthoringV1`
- Internal JSON keys unchanged

### Utils (`app/utils/`)

- `executable_contract.py` -> `service_definition.py`
- `compile_executable_contract_preview()` -> `compile_service_preview()`

### GraphQL (`app/graphql/`)

- `executable_contract.py` -> `service_definition.py`
- Type: `ExecutableContractType` -> `ServiceDefinitionType`
- Queries: `executable_contract()` -> `service_definition()`, `paginated_executable_contracts()` -> `paginated_service_definitions()`
- Mutations: `create_executable_contract()` -> `create_service_definition()`, `update_executable_contract()` -> `update_service_definition()`, `delete_executable_contract()` -> `delete_service_definition()`
- Compile: `compile_executable_contract_preview()` -> `compile_service_preview()`, `compile_executable_contract_preview_by_id()` -> `compile_service_preview_by_id()`
- Deployment: `deployContract()` -> `deployService()`

### Seed

- `seed_contracts.py` -> `seed_services.py`

## Frontend Changes

### Components (`src/features/`)

| Current | New |
|---------|-----|
| `contract-builder/` | `service-editor/` |
| `ContractBuilderPage.jsx` | `ServiceEditorPage.jsx` |
| `ContractListPage.jsx` | `ServiceListPage.jsx` |
| `ParamBuilderPanel.jsx` | `ParamEditorPanel.jsx` |
| `authoringAdapter.js` | `serviceAdapter.js` |
| `constants.js` | Updated query names |

Files that keep their names (content updated): `AuthoringJsonPanel.jsx`, `FormPreviewPanel.jsx`, `CompileOutputPanel.jsx`, `useDynamicForm.jsx`, `formUtils.js`, `builderUtils.js`, `fields/*`.

### Routes

- Path: `contracts` -> `services`
- Label: `"Contract Builder"` -> `"Services"`
- Full path: `/app/contracts` -> `/app/services`

### GraphQL Queries

All query/mutation names updated to match backend. Key changes:
- `GET_CONTRACTS_FOR_BINDING` -> `GET_SERVICES_FOR_BINDING`
- `DEPLOY_CONTRACT` -> `DEPLOY_SERVICE`
- `CREATE_EXECUTABLE_CONTRACT` -> `CREATE_SERVICE_DEFINITION`
- `COMPILE_EXECUTABLE_CONTRACT_PREVIEW` -> `COMPILE_SERVICE_PREVIEW`

### DeployModal

- "App Catalog" tab -> "Service Catalog"
- Variable names: `contract` -> `service`
- Labels: "Deploy from contract" -> "Deploy Service"

### i18n

Both `en/` and `zh/` translation files updated. All "Contract"/"contract" strings replaced with "Service"/"service" equivalents.

## Out of Scope

- No changes to the authoring JSON schema format or compilation logic
- No changes to systemd unit naming (`aurora-deploy-{id}`)
- No changes to the deployment workflow or lifecycle
- No UI redesign (layout, styling, interactions stay the same)

## Execution Order

1. Alembic migration (rename tables & columns)
2. Backend models & imports
3. Backend schemas (Pydantic)
4. Backend utils (compilation)
5. Backend GraphQL (types, resolvers, schema wiring)
6. Backend seed script
7. Frontend GraphQL queries
8. Frontend components (rename files, update imports)
9. Frontend routes & navigation
10. Frontend i18n translations
11. CLAUDE.md documentation update
