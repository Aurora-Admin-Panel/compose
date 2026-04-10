# Unified Deploy Service UX — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Merge the "Service Catalog" and "From Binding" tabs into a single unified deployable list, rename "Deploy Executable" → "Deploy Service", unify backend mutations, and extract binding management into a standalone reusable dialog.

**Architecture:** The DeployModal loses its tab UI and becomes a simple two-step flow: pick from a flat list (built-in services + bindings), then configure and deploy. The backend's two deploy mutations (`deployExecutable` + `deployService`) merge into one `deployService` that accepts either `serviceId` or `serviceBindingId`. The BindingModal gets `modalProps` support for pre-filtering by file or service, and entry points are added to ServiceListPage and FileCard.

**Tech Stack:** React 18, Apollo Client (GraphQL), Strawberry (Python GraphQL), SQLAlchemy, DaisyUI 5, i18next

---

## Task 1: Unify backend mutation — merge `deployExecutable` into `deployService`

**Files:**
- Modify: `backend/app/graphql/deployment.py:286-366` (remove `deploy_executable_resolver`)
- Modify: `backend/app/graphql/deployment.py:369-448` (update `deploy_service_resolver` to accept optional `service_binding_id`)
- Modify: `backend/app/graphql/schema.py:211-219` (remove `deploy_executable` field, update `deploy_service` field)

**Step 1: Update `deploy_service_resolver` to accept both `serviceId` and `serviceBindingId`**

In `backend/app/graphql/deployment.py`, replace `deploy_service_resolver` (lines 369-448) with a unified version that handles both paths. The resolver accepts `service_id: Optional[int] = None` and `service_binding_id: Optional[int] = None`, validates exactly one is provided, then resolves the service definition from whichever path was given.

```python
async def deploy_service_resolver(
    info: Info,
    server_ids: List[int],
    values: JSON,
    service_id: Optional[int] = None,
    service_binding_id: Optional[int] = None,
    port_id: Optional[int] = None,
) -> List[ServerDeployment]:
    """Deploy a service to one or more servers.

    Exactly one of service_id or service_binding_id must be provided.
    - service_id: deploy a built-in service directly (has source acquisition)
    - service_binding_id: deploy via a file-to-service binding
    """
    from tasks.deployment import deploy_executable_task

    if not service_id and not service_binding_id:
        raise ValueError("Either serviceId or serviceBindingId must be provided")
    if service_id and service_binding_id:
        raise ValueError("Provide only one of serviceId or serviceBindingId, not both")

    user = info.context["request"].state.user
    results = []
    pending_tasks = []

    async with async_db_session() as db:
        # Resolve service definition and validate
        if service_binding_id:
            binding = (await db.execute(
                select(DBServiceBinding).where(DBServiceBinding.id == service_binding_id)
            )).scalars().first()
            if not binding:
                raise ValueError(f"Service binding {service_binding_id} not found")
            service_def = (await db.execute(
                select(DBServiceDefinition).where(DBServiceDefinition.id == binding.service_id)
            )).scalars().first()
            if not service_def:
                raise ValueError(f"Service definition for binding {service_binding_id} not found")
        else:
            service_def = (await db.execute(
                select(DBServiceDefinition).where(DBServiceDefinition.id == service_id)
            )).scalars().first()
            if not service_def:
                raise ValueError(f"Service definition {service_id} not found")

        requires_port = _parse_requires_port(service_def.config_json)
        await _validate_port_for_deployment(db, port_id, server_ids, requires_port)

        for sid in server_ids:
            # Upsert: find existing by (binding+server) or (service+server)
            if service_binding_id:
                existing_stmt = select(DBServerDeployment).where(
                    DBServerDeployment.service_binding_id == service_binding_id,
                    DBServerDeployment.server_id == sid,
                )
            else:
                existing_stmt = select(DBServerDeployment).where(
                    DBServerDeployment.service_id == service_id,
                    DBServerDeployment.server_id == sid,
                )
            existing = (await db.execute(existing_stmt)).scalars().first()

            if existing:
                existing.values_json = values
                existing.status = DeploymentStatusEnum.PENDING
                existing.is_active = True
                existing.port_id = port_id
                deployment = existing
            else:
                deployment = DBServerDeployment(
                    service_binding_id=service_binding_id,
                    service_id=service_id,
                    server_id=sid,
                    values_json=values,
                    status=DeploymentStatusEnum.PENDING,
                    port_id=port_id,
                )
                db.add(deployment)

            await db.flush()

            log = DBDeploymentLog(
                deployment_id=deployment.id,
                action=DeploymentActionEnum.DEPLOY,
                status=DeploymentLogStatusEnum.PENDING,
                created_by_id=user.id if user else None,
            )
            db.add(log)
            await db.flush()

            pending_tasks.append((deployment.id, log.id, log))
            results.append(deployment)

        try:
            await db.commit()
        except IntegrityError:
            raise ValueError("Port is already in use by another deployment")

        for dep_id, log_id, log in pending_tasks:
            task_result = deploy_executable_task(dep_id, log_id)
            log.task_id = task_result.id
        await db.commit()

        for dep in results:
            await db.refresh(dep)

    return results
```

**Step 2: Remove `deploy_executable_resolver`**

Delete lines 286-366 (the old `deploy_executable_resolver` function) from `deployment.py`.

**Step 3: Update schema.py — remove `deploy_executable`, keep unified `deploy_service`**

In `backend/app/graphql/schema.py`:
- Remove the import of `deploy_executable_resolver` (line 34)
- Remove the `deploy_executable` field (lines 212-215)
- The `deploy_service` field (lines 216-219) stays as-is — strawberry auto-maps the new optional args

**Step 4: Verify backend builds**

Run: `docker-compose exec backend python -c "from app.graphql.schema import schema; print('OK')"`
Expected: `OK`

**Step 5: Commit**

```bash
git add backend/app/graphql/deployment.py backend/app/graphql/schema.py
git commit -m "refactor: unify deployExecutable + deployService into single mutation"
```

---

## Task 2: Update frontend GraphQL queries

**Files:**
- Modify: `frontend/src/queries/deployment.js:135-181` (remove `DEPLOY_EXECUTABLE`, update `DEPLOY_SERVICE`)

**Step 1: Update `DEPLOY_SERVICE` mutation to accept optional `serviceBindingId`**

In `frontend/src/queries/deployment.js`, replace the `DEPLOY_SERVICE` mutation (lines 159-181) with:

```javascript
export const DEPLOY_SERVICE = gql`
  mutation DeployService(
    $serviceId: Int
    $serviceBindingId: Int
    $serverIds: [Int!]!
    $values: JSON!
    $portId: Int
  ) {
    deployService(
      serviceId: $serviceId
      serviceBindingId: $serviceBindingId
      serverIds: $serverIds
      values: $values
      portId: $portId
    ) {
      id
      serviceBindingId
      serviceId
      serverId
      portId
      status
      createdAt
      updatedAt
    }
  }
`;
```

**Step 2: Remove `DEPLOY_EXECUTABLE` mutation**

Delete lines 135-157 (the `DEPLOY_EXECUTABLE` constant).

**Step 3: Commit**

```bash
git add frontend/src/queries/deployment.js
git commit -m "refactor: remove DEPLOY_EXECUTABLE query, unify into DEPLOY_SERVICE"
```

---

## Task 3: Rewrite DeployModal — unified flat list, no tabs

**Files:**
- Modify: `frontend/src/features/deployment/DeployModal.jsx` (full rewrite of step 1 UI)

**Step 1: Rewrite DeployModal**

Replace the entire content of `DeployModal.jsx`. Key changes:
- Remove tab state, `selectedFileId`, `selectedServiceId`, `handleCreateBindingAndContinue`
- Remove `CREATE_SERVICE_BINDING` and `DEPLOY_EXECUTABLE` imports
- Remove `GET_EXECUTABLE_FILES` import (no longer needed in deploy flow)
- Single state: `selectedItem` — either `{ type: "service", serviceId }` or `{ type: "binding", bindingId, service }`
- Step 1 renders a flat list combining `catalogServices` and `bindings`
- Step 2 unchanged (port selector + dynamic form + deploy)
- `handleDeploy` always calls `deployService` with appropriate args
- Empty state with links

```jsx
import { useState, useMemo, useCallback } from "react";
import { useTranslation } from "react-i18next";
import { useQuery, useMutation } from "@apollo/client";
import classNames from "classnames";
import { Package, Link as LinkIcon } from "lucide-react";
import DataLoading from "../DataLoading";
import useDynamicForm from "../service-editor/useDynamicForm";
import { serviceDefinitionToDynamicSchema } from "../service-editor/serviceAdapter";
import {
  GET_SERVICE_BINDINGS,
  GET_SERVICES_FOR_BINDING,
  GET_AVAILABLE_PORTS,
  DEPLOY_SERVICE,
} from "../../queries/deployment";
import ModalShell from "../ui/ModalShell";

const DeployModal = ({ modalProps, close, resolve }) => {
  const { t } = useTranslation();
  const serverId = modalProps?.serverId;

  const [step, setStep] = useState(1);
  const [selectedItem, setSelectedItem] = useState(null);
  const [selectedPortId, setSelectedPortId] = useState(null);
  const [formValues, setFormValues] = useState({});

  // Queries
  const { data: bindingsData, loading: bindingsLoading } =
    useQuery(GET_SERVICE_BINDINGS);
  const { data: servicesData, loading: servicesLoading } =
    useQuery(GET_SERVICES_FOR_BINDING);
  const { data: portsData, loading: portsLoading } = useQuery(GET_AVAILABLE_PORTS, {
    variables: { serverId },
    skip: !serverId,
  });

  const [deployService, { loading: deploying }] = useMutation(DEPLOY_SERVICE);

  const bindings = bindingsData?.serviceBindings ?? [];
  const services = servicesData?.serviceDefinitions ?? [];

  const catalogServices = useMemo(
    () => services.filter((s) => s.hasSource),
    [services]
  );

  // Build unified list: built-in services first, then bindings
  const deployableItems = useMemo(() => {
    const items = [];
    for (const s of catalogServices) {
      items.push({
        key: `service-${s.id}`,
        type: "service",
        serviceId: s.id,
        service: s,
        label: s.title,
        sublabel: `${s.serviceKey} v${s.version}`,
        isBuiltin: s.isBuiltin,
      });
    }
    for (const b of bindings) {
      items.push({
        key: `binding-${b.id}`,
        type: "binding",
        bindingId: b.id,
        service: b.service,
        label: `${b.file?.name || `File #${b.fileId}`}`,
        sublabel: b.service?.title || b.service?.serviceKey || `Service #${b.serviceId}`,
        fileVersion: b.file?.version,
      });
    }
    return items;
  }, [catalogServices, bindings]);

  const selectedService = selectedItem?.service ?? null;

  const formSchema = useMemo(() => {
    if (!selectedService?.configJson) return null;
    try {
      return serviceDefinitionToDynamicSchema(selectedService.configJson);
    } catch {
      return null;
    }
  }, [selectedService]);

  const requiresPort = selectedService?.configJson?.requiresPort !== false;

  const handleValuesChange = useCallback((values) => {
    setFormValues(values);
  }, []);

  const handleFormSubmit = useCallback((values) => {
    setFormValues(values);
  }, []);

  const handleSelectItem = (item) => {
    setSelectedItem(item);
    setSelectedPortId(null);
    setStep(2);
  };

  const handleDeploy = async () => {
    if (!serverId || !selectedItem) return;

    const variables = {
      serverIds: [serverId],
      values: formValues,
      portId: selectedPortId,
    };

    if (selectedItem.type === "service") {
      variables.serviceId = selectedItem.serviceId;
    } else {
      variables.serviceBindingId = selectedItem.bindingId;
    }

    await deployService({ variables });
    if (resolve) resolve(true);
    close();
  };

  const handleCancel = () => {
    if (resolve) resolve(false);
    close();
  };

  const handleBack = () => {
    setSelectedItem(null);
    setSelectedPortId(null);
    setFormValues({});
    setStep(1);
  };

  const isAnyLoading = bindingsLoading || servicesLoading;

  return (
    <ModalShell
      title={t("Deploy Service")}
      onClose={handleCancel}
      maxWidth="max-w-3xl"
    >
      {/* Steps indicator */}
      <ul className="steps steps-horizontal w-full">
        <li className={classNames("step", step >= 1 && "step-primary")}>
          {t("Select")}
        </li>
        <li className={classNames("step", step >= 2 && "step-primary")}>
          {t("Configure")}
        </li>
      </ul>

      {isAnyLoading ? (
        <div className="py-8">
          <DataLoading />
        </div>
      ) : (
        <>
          {/* Step 1: Pick a deployable item */}
          {step === 1 && (
            <div className="mt-4 space-y-4">
              {deployableItems.length === 0 ? (
                <div className="py-8 text-center">
                  <p className="text-sm opacity-60">{t("No services available")}</p>
                  <div className="mt-3 flex justify-center gap-2">
                    <a href="/app/services" className="btn btn-outline btn-sm">
                      {t("Browse Service Definitions")}
                    </a>
                    <a href="/app/files" className="btn btn-outline btn-sm">
                      {t("Upload a File")}
                    </a>
                  </div>
                </div>
              ) : (
                <div className="max-h-72 space-y-1 overflow-auto">
                  {deployableItems.map((item) => (
                    <div
                      key={item.key}
                      className="flex cursor-pointer items-center justify-between rounded-lg bg-base-300 px-3 py-2 hover:bg-primary/10"
                      onClick={() => handleSelectItem(item)}
                    >
                      <div className="flex items-center gap-2">
                        {item.type === "service" ? (
                          <Package size={14} className="opacity-40" />
                        ) : (
                          <LinkIcon size={14} className="opacity-40" />
                        )}
                        <span className="font-medium">{item.label}</span>
                        {item.type === "binding" && (
                          <>
                            <span className="opacity-40">→</span>
                            <span className="text-sm opacity-70">{item.sublabel}</span>
                          </>
                        )}
                        {item.type === "service" && (
                          <span className="badge badge-ghost badge-xs font-mono">
                            {item.sublabel}
                          </span>
                        )}
                        {item.isBuiltin && (
                          <span className="badge badge-info badge-xs">
                            {t("Built-in")}
                          </span>
                        )}
                      </div>
                      <div className="flex items-center gap-1">
                        {item.fileVersion && (
                          <span className="text-xs opacity-50">v{item.fileVersion}</span>
                        )}
                        {item.type === "service" && (
                          <span className="text-xs opacity-50">v{item.service?.version}</span>
                        )}
                      </div>
                    </div>
                  ))}
                </div>
              )}

              <div className="flex justify-end">
                <button className="btn btn-outline" onClick={handleCancel}>
                  {t("Cancel")}
                </button>
              </div>
            </div>
          )}

          {/* Step 2: Configure & deploy */}
          {step === 2 && (
            <div className="mt-4 space-y-4">
              {/* Port selector */}
              {requiresPort && (
                <div className="form-control w-full">
                  <label className="label">
                    <span className="label-text font-medium">{t("Port")}</span>
                  </label>
                  <select
                    className="select select-bordered w-full"
                    value={selectedPortId || ""}
                    onChange={(e) =>
                      setSelectedPortId(e.target.value ? Number(e.target.value) : null)
                    }
                  >
                    <option value="">{t("Select a port...")}</option>
                    {portsData?.availablePortsForDeployment?.map((port) => (
                      <option key={port.id} value={port.id}>
                        {t("Port")} {port.num}
                        {port.externalNum && port.externalNum !== port.num
                          ? ` → ${port.externalNum}`
                          : ""}
                      </option>
                    ))}
                  </select>
                  {!portsLoading &&
                    portsData?.availablePortsForDeployment?.length === 0 && (
                      <p className="text-sm text-error mt-1">
                        {t("No available ports on this server")}
                      </p>
                    )}
                </div>
              )}

              {/* Service form */}
              {formSchema && (
                <div>
                  <h4 className="mb-2 font-semibold text-sm">{t("Parameters")}</h4>
                  <div className="max-h-64 overflow-auto rounded-lg bg-base-300 p-3">
                    <ServiceValuesForm
                      formSchema={formSchema}
                      onSubmit={handleFormSubmit}
                      onValuesChange={handleValuesChange}
                    />
                  </div>
                </div>
              )}

              {/* Summary */}
              <div className="rounded-lg bg-base-300 p-4">
                <h4 className="mb-2 font-semibold text-sm">{t("Summary")}</h4>
                <div className="space-y-1 text-sm">
                  <div>
                    <span className="opacity-60">{t("Service")}:</span>{" "}
                    <span>
                      {selectedService?.title || selectedService?.serviceKey}
                    </span>
                  </div>
                  {selectedItem?.type === "binding" && (
                    <div>
                      <span className="opacity-60">{t("File")}:</span>{" "}
                      <span>{selectedItem.label}</span>
                    </div>
                  )}
                  {Object.keys(formValues).length > 0 && (
                    <div>
                      <span className="opacity-60">{t("Parameters")}:</span>
                      <pre className="mt-1 max-h-32 overflow-auto rounded bg-base-100 p-2 text-xs">
                        {JSON.stringify(formValues, null, 2)}
                      </pre>
                    </div>
                  )}
                </div>
              </div>

              <div className="flex justify-between">
                <button className="btn btn-outline btn-sm" onClick={handleBack}>
                  {t("Back")}
                </button>
                <button
                  className={classNames("btn btn-primary", { loading: deploying })}
                  onClick={handleDeploy}
                  disabled={deploying || (requiresPort && !selectedPortId)}
                >
                  {t("Deploy")}
                </button>
              </div>
            </div>
          )}
        </>
      )}
    </ModalShell>
  );
};

function ServiceValuesForm({ formSchema, onSubmit, onValuesChange }) {
  const { form } = useDynamicForm({
    schema: formSchema,
    onSubmit,
    onValuesChange,
  });
  return form;
}

export default DeployModal;
```

**Step 2: Commit**

```bash
git add frontend/src/features/deployment/DeployModal.jsx
git commit -m "refactor: DeployModal — unified flat list, no tabs"
```

---

## Task 4: Update BindingModal to support pre-filtering via `modalProps`

**Files:**
- Modify: `frontend/src/features/deployment/BindingModal.jsx`

**Step 1: Add `modalProps` support for `fileId` and `serviceId` pre-filtering**

Update `BindingModal` to accept `modalProps` and pass filter variables to `GET_SERVICE_BINDINGS`. When opened from FileCard, it receives `{ fileId }`. When opened from ServiceListPage, it receives `{ serviceId }`. The create form pre-selects the relevant dropdown.

```jsx
const BindingModal = ({ modalProps, close, resolve }) => {
```

Add to the query:
```jsx
const filterFileId = modalProps?.fileId ?? null;
const filterServiceId = modalProps?.serviceId ?? null;

const {
  data: bindingsData,
  loading: bindingsLoading,
  refetch,
} = useQuery(GET_SERVICE_BINDINGS, {
  variables: {
    ...(filterFileId && { fileId: filterFileId }),
    ...(filterServiceId && { serviceId: filterServiceId }),
  },
});
```

Initialize the create form dropdowns from `modalProps`:
```jsx
const [selectedFileId, setSelectedFileId] = useState(filterFileId ? String(filterFileId) : "");
const [selectedServiceId, setSelectedServiceId] = useState(filterServiceId ? String(filterServiceId) : "");
```

**Step 2: Commit**

```bash
git add frontend/src/features/deployment/BindingModal.jsx
git commit -m "feat: BindingModal accepts modalProps for pre-filtering by file or service"
```

---

## Task 5: Add "Manage Bindings" button to FileCard (for EXECUTABLE files)

**Files:**
- Modify: `frontend/src/features/file/FileCard.jsx:168-184` (add binding button)

**Step 1: Add a "Bindings" button for EXECUTABLE file type**

Import `LinkIcon` and add a conditional button in the actions section. When clicked, it opens the binding modal pre-filtered to that file.

Add to imports:
```jsx
import { Link as LinkIcon } from "lucide-react";
```

Note: `Link` is already potentially conflicting with react-router. Use `Link as LinkIcon` since the existing imports already show a rename pattern (`File as FileIcon`).

In the actions div (lines 169-184), add a "Bindings" button between Delete and Preview/Download, visible only for EXECUTABLE files:

```jsx
{/* Actions */}
<div className="mt-auto flex items-center gap-1.5 border-t border-base-content/[0.04] px-3 py-2">
  <button
    className="btn btn-ghost btn-sm flex-1 gap-1.5 text-xs opacity-60 transition-opacity hover:opacity-100"
    onClick={handleClickCancel}
  >
    <Trash2 size={13} />
    {t("Delete")}
  </button>
  {file.type === "EXECUTABLE" && (
    <button
      className="btn btn-ghost btn-sm flex-1 gap-1.5 text-xs opacity-60 transition-opacity hover:opacity-100"
      onClick={() => open("binding", { fileId: file.id })}
    >
      <LinkIcon size={13} />
      {t("Bindings")}
    </button>
  )}
  <button
    className="btn btn-primary btn-sm flex-1 gap-1.5 text-xs"
    onClick={handleCheck}
  >
    {isPreviewable ? <Eye size={13} /> : <Download size={13} />}
    {isPreviewable ? t("Preview") : t("Download")}
  </button>
</div>
```

**Step 2: Commit**

```bash
git add frontend/src/features/file/FileCard.jsx
git commit -m "feat: add Bindings button to executable file cards"
```

---

## Task 6: Add "Manage Bindings" button to ServiceListPage

**Files:**
- Modify: `frontend/src/features/service-editor/ServiceListPage.jsx:116-128` (add bindings button to actions column)

**Step 1: Import `useModal` and add button**

Add imports:
```jsx
import { Link as LinkIcon } from "lucide-react";
import { useModal } from "../../atoms/modal";
```

In the component, add:
```jsx
const { open } = useModal();
```

In the actions `<td>` (lines 116-128), add a "Bindings" button before the "Open" button:

```jsx
<td className="text-right">
  <button
    type="button"
    className="btn btn-ghost btn-xs"
    onClick={(e) => {
      e.stopPropagation();
      open("binding", { serviceId: item.id });
    }}
  >
    <LinkIcon size={14} />
    {t("Bindings")}
  </button>
  <button
    type="button"
    className="btn btn-ghost btn-xs"
    onClick={(e) => {
      e.stopPropagation();
      navigate(`/app/services/editor/${item.id}`);
    }}
  >
    <Pencil size={14} />
    {t("Open")}
  </button>
</td>
```

**Step 2: Commit**

```bash
git add frontend/src/features/service-editor/ServiceListPage.jsx
git commit -m "feat: add Bindings button to service definitions list"
```

---

## Task 7: Update i18n translation files

**Files:**
- Modify: `frontend/public/locales/en/translation.json`
- Modify: `frontend/public/locales/zh/translation.json`

**Step 1: Update English translations**

Changes:
- `"Deploy Executable"` → `"Deploy Service"` (keep the key, change value — actually the key IS the value in i18next default ns, so change the key)
- Add: `"No services available"`, `"Browse Service Definitions"`, `"Upload a File"`, `"Bindings"`
- The keys `"Service Catalog"` and `"From Binding"` can stay (no harm, may be used elsewhere) or be removed for cleanliness

Add these new keys to the English translation file:
```json
"Deploy Service": "Deploy Service",
"No services available": "No services available",
"Browse Service Definitions": "Browse Service Definitions",
"Upload a File": "Upload a File",
"Bindings": "Bindings"
```

Remove (or leave, low priority):
```json
"Deploy Executable": "Deploy Executable",
"Service Catalog": "Service Catalog",
"From Binding": "From Binding"
```

**Step 2: Update Chinese translations**

Add equivalent keys:
```json
"Deploy Service": "部署服务",
"No services available": "暂无可用服务",
"Browse Service Definitions": "浏览服务定义",
"Upload a File": "上传文件",
"Bindings": "绑定"
```

Remove the old keys that are no longer used.

**Step 3: Commit**

```bash
git add frontend/public/locales/en/translation.json frontend/public/locales/zh/translation.json
git commit -m "i18n: update translations for unified deploy service UX"
```

---

## Task 8: Clean up unused imports and dead code

**Files:**
- Modify: `frontend/src/queries/deployment.js` (verify `DEPLOY_EXECUTABLE` is gone, `GET_EXECUTABLE_FILES` may still be needed by BindingModal)
- Verify: `frontend/src/features/deployment/DeployModal.jsx` has no stale imports
- Verify: No other files import `DEPLOY_EXECUTABLE`

**Step 1: Search for remaining references to `DEPLOY_EXECUTABLE`**

Run: `grep -r "DEPLOY_EXECUTABLE" frontend/src/`

If any files still import it, update them.

**Step 2: Verify `GET_EXECUTABLE_FILES` is still used**

It's imported by `BindingModal.jsx` — keep it in `deployment.js`.

**Step 3: Commit if any changes**

```bash
git add -u frontend/
git commit -m "chore: clean up unused deployment imports"
```

---

## Task 9: Manual smoke test

**Step 1: Start the dev environment**

```bash
docker-compose up -d
docker-compose exec backend alembic upgrade head
cd frontend && npm run dev
```

**Step 2: Test the deploy flow**

1. Navigate to a server's deployment page
2. Click "Deploy" — verify modal title is "Deploy Service"
3. Verify a flat list appears with built-in services (Package icon, Built-in badge) and bindings (Link icon, "file → service" format)
4. Click a built-in service → configure step → deploy → verify deployment created
5. Click a binding → configure step → deploy → verify deployment created
6. Test empty state (if no services/bindings exist)

**Step 3: Test binding management**

1. Go to File Center → click "Bindings" on an executable file card → verify BindingModal opens pre-filtered to that file
2. Go to Service Definitions → click "Bindings" on a service row → verify BindingModal opens pre-filtered to that service
3. Create a binding, delete a binding — verify CRUD works

**Step 4: Commit all changes if any fixes needed**

```bash
git add -u
git commit -m "fix: smoke test fixes for unified deploy service"
```
