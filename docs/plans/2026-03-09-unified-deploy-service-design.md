# Unified Deploy Service UX — Design

**Date:** 2026-03-09
**Status:** Approved
**Goal:** Simplify the deployment UX by merging the "Service Catalog" and "From Binding" tabs into a single unified list of deployable items, rename "Deploy Executable" to "Deploy Service", and extract binding management into a standalone dialog.

---

## 1. Deploy Modal Redesign

**Title:** "Deploy Service" (was "Deploy Executable")

**Step 1 — Pick a deployable item.** A single flat list (no tabs) with two kinds of entries:

- **Built-in services** (`hasSource=true`): Show title, version, key badge, "Built-in" badge
- **Bindings** (file ↔ service pairs): Show as "filename → Service Title" with file type/version info

Empty state: "No services available" with links to "Browse Service Definitions" and "Upload a File".

**Step 2 — Configure & Deploy.** Same as today: port selector (if `requiresPort`), dynamic parameter form, deploy button.

## 2. Binding Management Dialog

A standalone reusable dialog for CRUD on file-to-service bindings. Accessible from two places:

- **Service Definitions page** — "Manage Bindings" button on a service definition, dialog opens pre-filtered to that service
- **File Center** — "Manage Bindings" button on executable file cards, dialog opens pre-filtered to that file

The dialog shows a table of bindings (file ↔ service), with create and delete actions. Same dialog component, different initial filter context.

## 3. Unified Backend Mutation

Merge `deployService` + `deployExecutable` into a single `deployService` mutation:

```graphql
deployService(
  serviceId: ID          # for built-in (direct)
  serviceBindingId: ID   # for binding-based
  serverIds: [ID!]!
  values: JSON
  portId: ID
): ServerDeployment
```

Exactly one of `serviceId` or `serviceBindingId` must be provided. The resolver dispatches to the same deployment logic either way.

## 4. i18n Changes

- "Deploy Executable" → "Deploy Service" (en + zh)
- Remove "Service Catalog" and "From Binding" tab keys
- Add empty state strings ("No services available", "Browse Service Definitions", "Upload a File")

## 5. What Gets Removed

- Tab UI in DeployModal (Service Catalog / From Binding)
- Inline binding creation inside the deploy modal
- `deployExecutable` mutation (replaced by unified `deployService`)
- `DEPLOY_EXECUTABLE` frontend mutation query

## 6. What Stays the Same

- Step 2 configure flow (port selector, dynamic form, deploy)
- Service definition schema and compilation
- Binding data model (file ↔ service M2M)
- `BindingModal` component (refactored into standalone reusable dialog)
