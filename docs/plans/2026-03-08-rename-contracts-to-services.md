# Rename Executable Contracts to Services — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Rename the "executable contract" abstraction to "service" throughout the entire stack — DB, backend, GraphQL API, frontend, i18n, docs.

**Architecture:** The underlying schema format, compilation logic, and deployment machinery stay the same. This is a naming/framing change. Tables are renamed via Alembic migration, Python/JS files are renamed and classes updated, GraphQL API surface changes, frontend routes change from `/app/contracts` to `/app/services`.

**Tech Stack:** Alembic (migration), SQLAlchemy 1.4 (models), Strawberry (GraphQL), React + Apollo Client (frontend), i18next (translations)

**Design doc:** `docs/plans/2026-03-08-rename-contracts-to-services-design.md`

---

### Task 1: Alembic Migration — Rename Tables and Columns

**Files:**
- Create: `backend/app/alembic/versions/<hash>_rename_contracts_to_services.py`

**Context:** Latest alembic head is `b7c9e2f4a1d3`. The migration renames tables and columns only — no data changes, no type changes.

**Step 1: Generate empty migration**

Run: `docker-compose exec backend alembic revision --autogenerate -m "rename contracts to services"`

Then replace the generated content with manual renames (autogenerate won't detect renames).

**Step 2: Write the migration**

```python
"""rename contracts to services

Revision ID: <generated>
Revises: b7c9e2f4a1d3
"""
from alembic import op

revision = "<generated>"
down_revision = "b7c9e2f4a1d3"
branch_labels = None
depends_on = None


def upgrade():
    # Rename tables
    op.rename_table("executable_contract", "service_definition")
    op.rename_table("file_contract_binding", "service_binding")

    # Rename columns in service_definition
    op.alter_column("service_definition", "contract_key", new_column_name="service_key")
    op.alter_column("service_definition", "schema_json", new_column_name="config_json")

    # Rename columns in service_binding
    op.alter_column("service_binding", "contract_id", new_column_name="service_id")

    # Rename columns in server_deployment
    op.alter_column("server_deployment", "contract_id", new_column_name="service_id")
    op.alter_column("server_deployment", "binding_id", new_column_name="service_binding_id")

    # Rename indexes
    op.execute("ALTER INDEX IF EXISTS ix_executable_contract_id RENAME TO ix_service_definition_id")
    op.execute("ALTER INDEX IF EXISTS ix_executable_contract_contract_key RENAME TO ix_service_definition_service_key")
    op.execute("ALTER INDEX IF EXISTS ix_file_contract_binding_id RENAME TO ix_service_binding_id")

    # Rename constraints
    op.execute("""
        ALTER TABLE service_definition
        RENAME CONSTRAINT _executable_contract_contract_key_version_uc
        TO _service_definition_service_key_version_uc
    """)
    op.execute("""
        ALTER TABLE service_binding
        RENAME CONSTRAINT _file_contract_binding_file_id_contract_id_uc
        TO _service_binding_file_id_service_id_uc
    """)


def downgrade():
    # Reverse column renames
    op.alter_column("server_deployment", "service_binding_id", new_column_name="binding_id")
    op.alter_column("server_deployment", "service_id", new_column_name="contract_id")
    op.alter_column("service_binding", "service_id", new_column_name="contract_id")
    op.alter_column("service_definition", "config_json", new_column_name="schema_json")
    op.alter_column("service_definition", "service_key", new_column_name="contract_key")

    # Reverse constraint renames
    op.execute("""
        ALTER TABLE service_binding
        RENAME CONSTRAINT _service_binding_file_id_service_id_uc
        TO _file_contract_binding_file_id_contract_id_uc
    """)
    op.execute("""
        ALTER TABLE service_definition
        RENAME CONSTRAINT _service_definition_service_key_version_uc
        TO _executable_contract_contract_key_version_uc
    """)

    # Reverse index renames
    op.execute("ALTER INDEX IF EXISTS ix_service_binding_id RENAME TO ix_file_contract_binding_id")
    op.execute("ALTER INDEX IF EXISTS ix_service_definition_service_key RENAME TO ix_executable_contract_contract_key")
    op.execute("ALTER INDEX IF EXISTS ix_service_definition_id RENAME TO ix_executable_contract_id")

    # Reverse table renames
    op.rename_table("service_binding", "file_contract_binding")
    op.rename_table("service_definition", "executable_contract")
```

**Step 3: Verify migration runs**

Run: `docker-compose exec backend alembic upgrade head`
Expected: Migration applies cleanly, tables renamed in DB.

**Step 4: Verify downgrade works**

Run: `docker-compose exec backend alembic downgrade -1`
Then: `docker-compose exec backend alembic upgrade head`
Expected: Both directions work.

**Step 5: Commit**

```bash
git add backend/app/alembic/versions/*rename_contracts_to_services*
git commit -m "feat: add migration to rename contracts to services"
```

---

### Task 2: Backend Models — Rename Files and Classes

**Files:**
- Rename: `backend/app/db/models/executable_contract.py` → `backend/app/db/models/service_definition.py`
- Rename: `backend/app/db/models/file_contract_binding.py` → `backend/app/db/models/service_binding.py`
- Modify: `backend/app/db/models/server_deployment.py`
- Modify: `backend/app/db/models/__init__.py`
- Modify: `backend/app/db/models/file.py`

**Step 1: Rename executable_contract.py → service_definition.py and update class**

New file `service_definition.py` — rename class to `ServiceDefinition`, table to `service_definition`, column `contract_key` → `service_key`, column `schema_json` → `config_json`. Update constraint name. Update relationship back_populates.

Key changes in the model:
- `__tablename__ = "service_definition"`
- `class ServiceDefinition(Base):`
- Column `service_key = Column(String, ...)` (was `contract_key`)
- Column `config_json = Column(MutableDict.as_mutable(JSON), ...)` (was `schema_json`)
- UniqueConstraint name: `_service_definition_service_key_version_uc`
- Relationship `file_bindings` → `bindings` with `back_populates="service"`
- Relationship `deployments` with `back_populates="service"`

**Step 2: Rename file_contract_binding.py → service_binding.py and update class**

New file `service_binding.py` — rename class to `ServiceBinding`, table to `service_binding`, column `contract_id` → `service_id`. Update constraint, relationships.

Key changes:
- `__tablename__ = "service_binding"`
- `class ServiceBinding(Base):`
- Column `service_id = Column(Integer, ForeignKey("service_definition.id"), ...)`
- UniqueConstraint name: `_service_binding_file_id_service_id_uc`
- Relationship `service = relationship("ServiceDefinition", back_populates="bindings")`
- Relationship `file` stays, update back_populates if needed

**Step 3: Update server_deployment.py**

- `binding_id` → `service_binding_id` with FK to `service_binding.id`
- `contract_id` → `service_id` with FK to `service_definition.id`
- Relationship `binding` → `service_binding = relationship("ServiceBinding", ...)`
- Relationship `contract` → `service = relationship("ServiceDefinition", ...)`

**Step 4: Update models/__init__.py**

```python
from .service_definition import ServiceDefinition
from .service_binding import ServiceBinding
# Remove old imports: ExecutableContract, FileContractBinding
# Update __all__ list
```

**Step 5: Update file.py**

- Relationship `contract_bindings` → `service_bindings`
- `back_populates` references updated

**Step 6: Delete old files**

Remove `executable_contract.py` and `file_contract_binding.py` from models dir.

**Step 7: Verify imports work**

Run: `docker-compose exec backend python3 -c "from app.db.models import ServiceDefinition, ServiceBinding; print('OK')"`
Expected: OK

**Step 8: Commit**

```bash
git add backend/app/db/models/
git commit -m "refactor: rename contract models to service models"
```

---

### Task 3: Backend Schemas & Utils — Rename Pydantic Models and Compilation

**Files:**
- Rename: `backend/app/db/schemas/executable_contract.py` → `backend/app/db/schemas/service_definition.py`
- Rename: `backend/app/utils/executable_contract.py` → `backend/app/utils/service_definition.py`

**Step 1: Rename and update schema file**

Rename `executable_contract.py` to `service_definition.py`. Update class names:
- `ExecutableContractAuthoringV1` → `ServiceDefinitionAuthoringV1`
- `ContractUI` → `ServiceUI`
- `ContractCompileError` stays in utils (rename there)

**Important:** Do NOT change the internal JSON field names (`contractKey`, `exec`, `params`). These are part of the stored schema format. The Pydantic model field name stays `contractKey` — it's the JSON key, not a Python naming concern.

Keep `ALLOWED_TEMPLATE_VARS = {"jobId", "contractKey", "paramKey"}` — these are template variables in stored JSON.

**Step 2: Rename and update utils file**

Rename `executable_contract.py` to `service_definition.py`. Update:
- `ContractCompileError` → `ServiceCompileError`
- `compile_executable_contract_preview()` → `compile_service_preview()`
- Update imports from new schema path

**Step 3: Delete old files**

Remove old `executable_contract.py` from both schemas/ and utils/.

**Step 4: Verify**

Run: `docker-compose exec backend python3 -c "from app.db.schemas.service_definition import ServiceDefinitionAuthoringV1; print('OK')"`
Run: `docker-compose exec backend python3 -c "from app.utils.service_definition import compile_service_preview; print('OK')"`
Expected: Both print OK

**Step 5: Commit**

```bash
git add backend/app/db/schemas/ backend/app/utils/
git commit -m "refactor: rename contract schemas and utils to service"
```

---

### Task 4: Backend GraphQL — Rename Types and Resolvers

**Files:**
- Rename: `backend/app/graphql/executable_contract.py` → `backend/app/graphql/service_definition.py`
- Modify: `backend/app/graphql/deployment.py`
- Modify: `backend/app/graphql/schema.py`

**Step 1: Rename and update GraphQL service_definition.py**

Rename file. Update all names:
- Import `ServiceDefinition as DBServiceDefinition` (was `ExecutableContract as DBExecutableContract`)
- Import `ServiceDefinitionAuthoringV1` (was `ExecutableContractAuthoringV1`)
- Import `compile_service_preview` (was `compile_executable_contract_preview`)
- Type class: `class ServiceDefinitionType:` (was `ExecutableContract`)
- Field: `service_key` (was `contract_key`)
- Field: `config_json` (was `schema_json`)
- Resolver: `get_service_definition` (was `get_executable_contract`)
- Resolver: `get_service_definitions` (was `get_executable_contracts`)
- Resolver: `get_paginated_service_definitions` (was `get_paginated_executable_contracts`)
- Mutation: `create_service_definition` (was `create_executable_contract`)
- Mutation: `update_service_definition` (was `update_executable_contract`)
- Mutation: `delete_service_definition` (was `delete_executable_contract`)
- Resolver: `compile_service_preview_resolver` (was `compile_executable_contract_preview_resolver`)
- Resolver: `compile_service_preview_by_id_resolver` (was `compile_executable_contract_preview_by_id_resolver`)
- Internal helper: `_compact_config_json` (was `_compact_schema_json`)
- Error message: `"Service definition '{id}' not found"` (was `"Executable contract..."`)

**Step 2: Update deployment.py**

- Import `ServiceBinding as DBServiceBinding` (was `FileContractBinding as DBFileContractBinding`)
- Import `ServiceDefinition as DBServiceDefinition` (was `ExecutableContract as DBExecutableContract`)
- Type class: `class ServiceBindingType:` (was `FileContractBinding`)
- Field: `service_id` (was `contract_id`)
- Resolver: `get_service_bindings` (was `get_file_contract_bindings`)
- Mutation: `create_service_binding` (was `create_file_contract_binding`)
- Mutation: `delete_service_binding` (was `delete_file_contract_binding`)
- Mutation: `deploy_service` (was `deploy_contract`)
- ServerDeployment field references: `service_binding_id`, `service_id` (were `binding_id`, `contract_id`)
- Method: `service_title()` (was `contract_title()`)
- Relationship access: `.service_binding` and `.service` (were `.binding` and `.contract`)

**Step 3: Update schema.py**

Update all imports and field wiring:
- Import from `service_definition` module instead of `executable_contract`
- Import `ServiceBindingType` instead of `FileContractBinding`
- Query fields: `service_definition`, `service_definitions`, `paginated_service_definitions`, `service_bindings`
- Mutation fields: `compile_service_preview`, `compile_service_preview_by_id`, `create_service_definition`, `update_service_definition`, `delete_service_definition`, `create_service_binding`, `delete_service_binding`, `deploy_service`

**Step 4: Delete old file**

Remove `backend/app/graphql/executable_contract.py`.

**Step 5: Verify GraphQL schema generates**

Run: `docker-compose exec backend python3 -c "from app.graphql.schema import schema; print(schema.as_str()[:200])"`
Expected: Schema prints without error.

**Step 6: Commit**

```bash
git add backend/app/graphql/
git commit -m "refactor: rename contract GraphQL types to service"
```

---

### Task 5: Backend Tasks & Seed — Update Deployment Tasks and Seed Script

**Files:**
- Modify: `backend/tasks/deployment.py`
- Rename: `backend/app/seed_contracts.py` → `backend/app/seed_services.py`

**Step 1: Update tasks/deployment.py**

- Import `ServiceBinding` instead of `FileContractBinding`
- Import `ServiceDefinition` instead of `ExecutableContract`
- Import `compile_service_preview` from `app.utils.service_definition`
- Update all variable references: `binding` → `service_binding` where it references the model
- Update `contract` → `service` where it references `ServiceDefinition`
- Update function calls from `compile_executable_contract_preview` to `compile_service_preview`

**Step 2: Rename and update seed script**

Rename `seed_contracts.py` to `seed_services.py`. Update:
- Docstring: "Seed built-in service definitions"
- Import: `from app.db.models import ServiceDefinition`
- Variable: `BUILTIN_SERVICES` (was `BUILTIN_CONTRACTS`)
- All references to `ExecutableContract` → `ServiceDefinition`
- Column references: `service_key` (was `contract_key`), `config_json` (was `schema_json`)

**Important:** The JSON payload inside each service definition still uses `contractKey` as the JSON field name — do NOT change the stored JSON structure.

**Step 3: Delete old seed file**

Remove `backend/app/seed_contracts.py`.

**Step 4: Verify seed runs**

Run: `docker-compose exec backend python3 app/seed_services.py`
Expected: Seeds run without error.

**Step 5: Commit**

```bash
git add backend/tasks/deployment.py backend/app/seed_services.py
git rm backend/app/seed_contracts.py
git commit -m "refactor: rename contract references in tasks and seed script"
```

---

### Task 6: Backend Tests — Update Test File

**Files:**
- Rename: `backend/tests/executable_contract_preview_test.py` → `backend/tests/service_preview_test.py`

**Step 1: Rename and update test file**

- Rename file to `service_preview_test.py`
- Update imports to use new module paths
- Update function references from `compile_executable_contract_preview` to `compile_service_preview`
- Update variable names as needed (test function names can stay descriptive of what they test)

**Step 2: Run tests**

Run: `docker-compose exec backend pytest tests/service_preview_test.py -v`
Expected: All tests pass.

**Step 3: Commit**

```bash
git add backend/tests/
git rm backend/tests/executable_contract_preview_test.py
git commit -m "refactor: rename contract test to service test"
```

---

### Task 7: Frontend GraphQL Queries — Update Query Definitions

**Files:**
- Modify: `frontend/src/features/contract-builder/constants.js` (will be moved in Task 8, but update content first)
- Modify: `frontend/src/queries/deployment.js`

**Step 1: Update constants.js query names**

All GraphQL operation names and field references must match the new backend API:
- `LIST_EXECUTABLE_CONTRACTS` → `LIST_SERVICE_DEFINITIONS` (query field: `paginatedServiceDefinitions` or `serviceDefinitions`)
- `CREATE_EXECUTABLE_CONTRACT` → `CREATE_SERVICE_DEFINITION` (mutation: `createServiceDefinition`)
- `UPDATE_EXECUTABLE_CONTRACT` → `UPDATE_SERVICE_DEFINITION` (mutation: `updateServiceDefinition`)
- `COMPILE_EXECUTABLE_CONTRACT_PREVIEW` → `COMPILE_SERVICE_PREVIEW` (mutation: `compileServicePreview`)
- `COMPILE_EXECUTABLE_CONTRACT_PREVIEW_BY_ID` → `COMPILE_SERVICE_PREVIEW_BY_ID` (mutation: `compileServicePreviewById`)
- `DEFAULT_CONTRACT_TEMPLATE` → `DEFAULT_SERVICE_TEMPLATE`

**Step 2: Update deployment.js query names**

- `GET_FILE_CONTRACT_BINDINGS` → `GET_SERVICE_BINDINGS` (query field: `serviceBindings`)
- `CREATE_FILE_CONTRACT_BINDING` → `CREATE_SERVICE_BINDING` (mutation: `createServiceBinding`)
- `DELETE_FILE_CONTRACT_BINDING` → `DELETE_SERVICE_BINDING` (mutation: `deleteServiceBinding`)
- `GET_CONTRACTS_FOR_BINDING` → `GET_SERVICES_FOR_BINDING` (query field: `serviceDefinitions`)
- `DEPLOY_CONTRACT` → `DEPLOY_SERVICE` (mutation: `deployService`)
- All field references: `contractId` → `serviceId`, `contractKey` → `serviceKey`, `schemaJson` → `configJson`, `bindingId` → `serviceBindingId`
- Note: `DEPLOY_EXECUTABLE` stays (it deploys via binding, name is fine). Update its `bindingId` param to `serviceBindingId` if the backend changed.

**Step 3: Commit**

```bash
git add frontend/src/features/contract-builder/constants.js frontend/src/queries/deployment.js
git commit -m "refactor: rename contract GraphQL queries to service"
```

---

### Task 8: Frontend Components — Rename Contract Builder to Service Editor

**Files:**
- Rename directory: `frontend/src/features/contract-builder/` → `frontend/src/features/service-editor/`
- Rename: `ContractBuilderPage.jsx` → `ServiceEditorPage.jsx`
- Rename: `ContractListPage.jsx` → `ServiceListPage.jsx`
- Rename: `ParamBuilderPanel.jsx` → `ParamEditorPanel.jsx`
- Rename: `authoringAdapter.js` → `serviceAdapter.js`
- Modify: All other files in the directory (update imports)

**Step 1: Rename directory**

```bash
cd frontend/src/features
mv contract-builder service-editor
```

**Step 2: Rename files**

```bash
cd service-editor
mv ContractBuilderPage.jsx ServiceEditorPage.jsx
mv ContractListPage.jsx ServiceListPage.jsx
mv ParamBuilderPanel.jsx ParamEditorPanel.jsx
mv authoringAdapter.js serviceAdapter.js
```

**Step 3: Update ServiceEditorPage.jsx**

- Function name: `ServiceEditorPage` (was `ContractBuilderPage`)
- Import: `serviceDefinitionToDynamicSchema` from `./serviceAdapter` (was `authoringContractToDynamicSchema` from `./authoringAdapter`)
- Import: renamed query constants from `./constants`
- Variable names: `contracts` → `services`, `selectedContract` → `selectedService`, etc.
- Panel component imports: `ParamEditorPanel` (was `ParamBuilderPanel`)

**Step 4: Update ServiceListPage.jsx**

- Function name: `ServiceListPage`
- Import: renamed query constants
- Variable references updated

**Step 5: Update ParamEditorPanel.jsx**

- Function name: `ParamEditorPanel`
- Any "contract" labels in JSX → "service"

**Step 6: Update serviceAdapter.js**

- Function name: `serviceDefinitionToDynamicSchema` (was `authoringContractToDynamicSchema`)
- Internal references updated

**Step 7: Update remaining files**

- `AuthoringJsonPanel.jsx`: Update any "contract" text
- `FormPreviewPanel.jsx`: Update any "contract" references
- `CompileOutputPanel.jsx`: Update any "contract" references
- `constants.js`: Already updated in Task 7
- `formUtils.js`, `builderUtils.js`: Update if they reference "contract"

**Step 8: Update the directory index export**

If there's an `index.js` or the directory default export points to the main page, update to export `ServiceEditorPage`.

**Step 9: Commit**

```bash
git add frontend/src/features/service-editor/
git commit -m "refactor: rename contract-builder to service-editor"
```

---

### Task 9: Frontend Deployment — Update DeployModal and BindingModal

**Files:**
- Modify: `frontend/src/features/deployment/DeployModal.jsx`
- Modify: `frontend/src/features/deployment/BindingModal.jsx`

**Step 1: Update DeployModal.jsx**

- Import from `../service-editor/serviceAdapter` (was `../contract-builder/authoringAdapter`)
- Import from `../service-editor/useDynamicForm` (was `../contract-builder/useDynamicForm`)
- Import renamed query constants from `../../queries/deployment`
- Variable renames:
  - `selectedBindingId` → `selectedServiceBindingId`
  - `selectedContractId` → `selectedServiceId`
  - `selectedCatalogContractId` → `selectedCatalogServiceId`
  - `contracts` → `services`
  - `catalogContracts` → `catalogServices`
  - `bindings` → `serviceBindings`
- Tab label: "App Catalog" → "Service Catalog"
- Function calls updated to use renamed mutations

**Step 2: Update BindingModal.jsx**

- Import renamed query constants
- Variable renames: `selectedContractId` → `selectedServiceId`, `contracts` → `services`, `bindings` → `serviceBindings`
- Modal title: "File-Contract Bindings" → "Service Bindings"
- Labels and references updated

**Step 3: Commit**

```bash
git add frontend/src/features/deployment/
git commit -m "refactor: rename contract references in deployment UI"
```

---

### Task 10: Frontend Routes & App — Update Routing

**Files:**
- Modify: `frontend/src/routes.js`
- Modify: `frontend/src/App.jsx`

**Step 1: Update routes.js**

```javascript
// Was: key: "contracts", path: "contracts", fullPath: "/app/contracts", labelKey: "Schemas"
{
  key: "services",
  path: "services",
  fullPath: "/app/services",
  area: "app",
  labelKey: "Services",
  icon: /* keep same icon or update */,
  permissions: ["admin", "ops"],
  nav: true,
},
// Was: key: "contractBuilder", path: "contracts/builder"
{
  key: "serviceEditor",
  path: "services/editor",
  fullPath: "/app/services/editor",
  area: "app",
  permissions: ["admin", "ops"],
},
// Was: key: "contractBuilderById", path: "contracts/builder/:contractId"
{
  key: "serviceEditorById",
  path: "services/editor/:serviceId",
  fullPath: "/app/services/editor/:serviceId",
  area: "app",
  permissions: ["admin", "ops"],
},
```

**Step 2: Update App.jsx**

- Lazy import: `const ServiceEditorPage = lazy(() => import("./features/service-editor"));`
- Lazy import: `const ServiceListPage = lazy(() => import("./features/service-editor/ServiceListPage"));`
- Route elements: Update `routeMap.services.path`, `routeMap.serviceEditor.path`, `routeMap.serviceEditorById.path`
- Navigate default: `routeMap.serviceEditor.fullPath`

**Step 3: Update any other navigation references**

Search for `routeMap.contracts` or `routeMap.contractBuilder` in other components and update.

**Step 4: Verify routes work**

Run: `cd frontend && npm run build`
Expected: Build succeeds.

**Step 5: Commit**

```bash
git add frontend/src/routes.js frontend/src/App.jsx
git commit -m "refactor: rename contract routes to service routes"
```

---

### Task 11: Frontend i18n — Update Translations

**Files:**
- Modify: `frontend/public/locales/en/translation.json`
- Modify: `frontend/public/locales/zh/translation.json`

**Step 1: Update English translations**

Key changes:
- `"Schemas"` → `"Services"`
- `"Command Schemas"` → `"Service Definitions"`
- `"Schema Builder"` → `"Service Editor"`
- `"Contracts"` → `"Services"`
- `"No contracts yet"` → `"No services yet"`
- `"Contract Key"` → `"Service Key"`
- `"Contract"` → `"Service"`
- `"File-Contract Bindings"` → `"Service Bindings"`
- `"App Catalog"` → `"Service Catalog"`
- `"Preview Auto Compiles By Stored Contract"` → `"Preview Auto Compiles By Stored Service"`
- Add any new keys needed

**Step 2: Update Chinese translations**

Mirror the English changes:
- `"方案"` stays (good word for "Service Definition" in this context) or update to `"服务"`
- `"方案列表"` → `"服务列表"`
- `"还没有方案"` → `"还没有服务"`
- `"方案标识"` → `"服务标识"`
- `"文件-合约绑定"` → `"服务绑定"`

**Step 3: Commit**

```bash
git add frontend/public/locales/
git commit -m "refactor: rename contract i18n strings to service"
```

---

### Task 12: Documentation — Update CLAUDE.md and Memory

**Files:**
- Modify: `CLAUDE.md`
- Modify: Memory files if needed

**Step 1: Update CLAUDE.md**

The "Executable Contract System" section (around line 109) needs to be renamed to "Service Definition System" with all internal references updated:
- "Executable Contract" → "Service Definition"
- "contract builder" → "service editor"
- File paths updated to new locations
- `authoringAdapter.js` → `serviceAdapter.js`
- `ContractBuilderPage.jsx` → `ServiceEditorPage.jsx`
- `features/contract-builder/` → `features/service-editor/`
- `deployContract` → `deployService`

**Step 2: Update deploy/ submodule docs (optional)**

If working within the deploy submodule:
- `deploy/executable-contract-schema.en.md` → rename or update title
- `deploy/executable-contract-schema.zh.md` → rename or update title

This is optional and can be done separately since deploy/ is a submodule.

**Step 3: Commit**

```bash
git add CLAUDE.md
git commit -m "docs: update CLAUDE.md for service definition rename"
```

---

### Task 13: Integration Verification

**Step 1: Run all backend tests**

Run: `docker-compose exec backend pytest -v`
Expected: All tests pass.

**Step 2: Build frontend**

Run: `cd frontend && npm run build`
Expected: Build succeeds with no errors.

**Step 3: Full stack smoke test**

Run: `docker-compose up -d && docker-compose exec backend alembic upgrade head`

Verify:
1. Navigate to `/app/services` — should show the service list
2. Navigate to `/app/services/editor` — should load the service editor
3. Open deploy modal — "Service Catalog" tab should work
4. Seed services: `docker-compose exec backend python3 app/seed_services.py`

**Step 4: Final commit if any fixups needed**

```bash
git add -A
git commit -m "fix: address integration issues from contract-to-service rename"
```

---

## Execution Notes

- **Order matters**: Tasks 1-6 (backend) must complete before Tasks 7-11 (frontend), since the GraphQL API surface changes.
- **Tasks 7-11** (frontend) can potentially be parallelized since they touch different files, but Task 8 (directory rename) should come before Task 9 (DeployModal imports from the new path).
- **JSON schema format**: Never change `contractKey` or other field names inside stored JSON payloads. The Pydantic model field stays `contractKey` — only Python class names, table names, and column names change.
- **Submodule changes**: The `frontend/` directory is a git submodule. Commits to frontend files are commits within the submodule.
