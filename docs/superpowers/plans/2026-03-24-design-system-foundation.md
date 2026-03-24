# Design System Foundation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Establish radix-nova design language (font, bg-muted surface, shared patterns) and redesign the Servers page as proof-of-concept.

**Architecture:** Install Noto Sans font, apply bg-muted page surface, update shared UI patterns (PageHeader, EmptyState, ThemedSuspense), then rewrite ServerList as a table-in-card with inline Progress bars for metrics. Delete unused card/stat sub-components.

**Tech Stack:** React 19, Tailwind CSS 4, shadcn/ui radix-nova, Recharts, Apollo Client

**Spec:** `docs/superpowers/specs/2026-03-24-design-system-foundation-design.md`

---

## File Map

### Modified files
- `frontend/src/index.css` — Font import + `--font-sans` variable
- `frontend/src/Layout.tsx:262-267` — `bg-muted`, `max-w-7xl`, `p-5 md:p-6`
- `frontend/src/features/ui/PageHeader.tsx` — Simplified to smaller typography
- `frontend/src/features/ui/EmptyState.tsx` — Rewrite to use `Empty` component
- `frontend/src/features/ThemedSuspense.tsx` — Centered `Spinner` on `bg-muted`
- `frontend/src/features/server/ServerList.tsx` — Full rewrite as table-in-card
- `frontend/src/features/server/ServerInfoModal.tsx` — Restyle to Field/FieldGroup pattern
- `frontend/src/features/server/ServerContainer.tsx:9` — Fix React Router v6 matchPath
- `frontend/src/features/file/FileCenter.tsx` — Rewrite as table-in-card
- `frontend/src/features/deployment/DeploymentList.tsx` — Rewrite as table-in-card

### Deleted files
- `frontend/src/features/server/ServerCard.tsx`
- `frontend/src/features/server/ServerRow.tsx`
- `frontend/src/features/server/ServerStat.tsx`
- `frontend/src/features/server/ServerPortsStat.tsx`
- `frontend/src/features/server/ServerTrafficStat.tsx`
- `frontend/src/features/server/ServerSSHStat.tsx`
- `frontend/src/hooks/useServerItem.ts`
- `frontend/src/features/file/FileCard.tsx`
- `frontend/src/features/file/FileRow.tsx`

---

## Task 1: Install font and update global styles

**Files:**
- Modify: `frontend/package.json`
- Modify: `frontend/src/index.css`

- [ ] **Step 1: Install Noto Sans font package**

```bash
cd /home/lei/workspace/created/aurora/frontend && npm install @fontsource-variable/noto-sans
```

- [ ] **Step 2: Add font import to `index.css`**

In `frontend/src/index.css`, add the font import after the existing imports (after line 3 `@import "shadcn/tailwind.css";`):

```css
@import "@fontsource-variable/noto-sans";
```

- [ ] **Step 3: Update `--font-sans` in the `@theme inline` block**

Replace line 7:
```diff
- --font-sans: var(--font-sans);
+ --font-sans: 'Noto Sans Variable', sans-serif;
```

Leave `--font-heading: var(--font-sans);` as-is — it correctly inherits the new font.

- [ ] **Step 4: Type-check**

```bash
cd /home/lei/workspace/created/aurora/frontend && npx tsc --noEmit
```

- [ ] **Step 5: Commit (inside frontend submodule)**

```bash
cd /home/lei/workspace/created/aurora/frontend
git add package.json package-lock.json src/index.css
git commit -m "style: add Noto Sans Variable font and update --font-sans theme token"
```

---

## Task 2: Update Layout — bg-muted surface and container

**Files:**
- Modify: `frontend/src/Layout.tsx:262-267`

- [ ] **Step 1: Add `bg-muted` to content area and update container**

In `frontend/src/Layout.tsx`, find the content wrapper (around line 262):

```diff
- <div className="flex-1 overflow-auto">
+ <div className="flex-1 overflow-auto bg-muted">
```

And update the container div:

```diff
- <div className="container p-4 md:p-6">
+ <div className="container max-w-7xl p-5 md:p-6">
```

- [ ] **Step 2: Commit**

```bash
cd /home/lei/workspace/created/aurora/frontend
git add src/Layout.tsx
git commit -m "style: apply bg-muted page surface and max-w-7xl container"
```

---

## Task 3: Update shared UI patterns

**Files:**
- Modify: `frontend/src/features/ui/PageHeader.tsx`
- Modify: `frontend/src/features/ui/EmptyState.tsx`
- Modify: `frontend/src/features/ThemedSuspense.tsx`

- [ ] **Step 1: Rewrite `PageHeader.tsx`**

Read the current file first. Replace it with a simplified version that is backward-compatible (keeps `onAdd`/`addLabel` for existing callers, adds `children` for new callers):

```tsx
import { Plus } from "lucide-react";
import { cn } from "@/lib/utils";
import { Button } from "@/components/ui/button";

interface PageHeaderProps {
  title: string;
  description?: string;
  /** New: pass action buttons as children */
  children?: React.ReactNode;
  /** Legacy: renders a "+ label" button */
  onAdd?: () => void;
  /** Legacy: label for the add button */
  addLabel?: string;
  className?: string;
}

export function PageHeader({ title, description, children, onAdd, addLabel, className }: PageHeaderProps) {
  return (
    <div className={cn("flex items-start justify-between gap-4", className)}>
      <div className="space-y-1">
        <h1 className="text-lg font-semibold">{title}</h1>
        {description && (
          <p className="text-sm text-muted-foreground">{description}</p>
        )}
      </div>
      <div className="flex items-center gap-2 shrink-0">
        {children}
        {onAdd && (
          <Button onClick={onAdd}>
            <Plus className="size-4 mr-1" />
            {addLabel}
          </Button>
        )}
      </div>
    </div>
  );
}

export default PageHeader;
```

Key changes: title goes from `text-2xl font-extrabold` to `text-lg font-semibold`. Keeps `onAdd`/`addLabel` for backward compat with FileCenter and ServerPorts (out of scope). New code uses `children` pattern. No Card wrapping.

- [ ] **Step 2: Rewrite `EmptyState.tsx`**

Read the current file first. Replace it to use the `Empty` component. Keep backward-compatible props (`icon`, `action`, `description`) for callers outside scope (FileCenter, PortUsersCard):

```tsx
import { Empty, EmptyHeader, EmptyMedia, EmptyTitle, EmptyDescription, EmptyContent } from "@/components/ui/empty";

interface EmptyStateProps {
  title: string;
  /** Primary description text */
  message?: string;
  /** Legacy alias for message */
  description?: string;
  /** Legacy: icon element rendered above the title */
  icon?: React.ReactNode;
  /** Legacy: action element rendered below the description */
  action?: React.ReactNode;
  /** New: pass action buttons as children */
  children?: React.ReactNode;
  className?: string;
}

export function EmptyState({ title, message, description, icon, action, children, className }: EmptyStateProps) {
  const desc = message || description;
  return (
    <Empty className={className}>
      {icon && <EmptyMedia>{icon}</EmptyMedia>}
      <EmptyHeader>
        <EmptyTitle>{title}</EmptyTitle>
        {desc && <EmptyDescription>{desc}</EmptyDescription>}
      </EmptyHeader>
      {(children || action) && (
        <EmptyContent>{children || action}</EmptyContent>
      )}
    </Empty>
  );
}

export default EmptyState;
```

- [ ] **Step 3: Update `ThemedSuspense.tsx`**

Read the current file first. Replace with centered Spinner on bg-muted:

```tsx
import { Spinner } from "@/components/ui/spinner";

export default function ThemedSuspense() {
  return (
    <div className="flex h-full min-h-[200px] items-center justify-center">
      <Spinner className="size-6" />
    </div>
  );
}
```

- [ ] **Step 4: Type-check**

```bash
cd /home/lei/workspace/created/aurora/frontend && npx tsc --noEmit 2>&1 | head -20
```

Note: There may be type errors in files that consume the old PageHeader/EmptyState APIs (e.g., `onAdd` prop removed from PageHeader). Check which callers break and update them if they are in the server feature area (in scope). For callers in other pages (files, services, etc.), either keep backward compatibility by accepting both prop styles, or leave the errors for subsequent sub-projects.

If backward compatibility is needed, add optional `onAdd`/`addLabel` props to PageHeader that render a Button with Plus icon when provided (preserving the old API while also supporting children).

- [ ] **Step 5: Commit**

```bash
cd /home/lei/workspace/created/aurora/frontend
git add src/features/ui/PageHeader.tsx src/features/ui/EmptyState.tsx src/features/ThemedSuspense.tsx
git commit -m "style: update PageHeader, EmptyState, and ThemedSuspense to radix-nova patterns"
```

---

## Task 4: Fix ServerContainer matchPath

**Files:**
- Modify: `frontend/src/features/server/ServerContainer.tsx`

- [ ] **Step 1: Read the file and fix matchPath call**

The file uses React Router v5 `matchPath` API. Fix to v6:

```diff
- const isServerListRoot = matchPath({ path: "/app/servers", exact: true }, location.pathname);
+ const isServerListRoot = matchPath("/app/servers", location.pathname);
```

Or if the pattern uses different syntax, read the file first and adjust accordingly.

- [ ] **Step 2: Commit**

```bash
cd /home/lei/workspace/created/aurora/frontend
git add src/features/server/ServerContainer.tsx
git commit -m "fix: update ServerContainer to use React Router v6 matchPath API"
```

---

## Task 5: Rewrite ServerList as table-in-card

**Files:**
- Rewrite: `frontend/src/features/server/ServerList.tsx`
- Delete: `frontend/src/features/server/ServerCard.tsx`
- Delete: `frontend/src/features/server/ServerRow.tsx`
- Delete: `frontend/src/features/server/ServerStat.tsx`
- Delete: `frontend/src/features/server/ServerPortsStat.tsx`
- Delete: `frontend/src/features/server/ServerTrafficStat.tsx`
- Delete: `frontend/src/features/server/ServerSSHStat.tsx`
- Delete: `frontend/src/hooks/useServerItem.ts`

This is the largest task. Read ALL files being deleted first to understand the logic that needs to be preserved.

- [ ] **Step 1: Read all server component files**

Read these files to understand the data flow and logic:
- `ServerList.tsx` — The parent. Has GraphQL query for paginated servers, subscription for metrics, state for modal.
- `ServerCard.tsx` — Renders a single server as a card.
- `ServerRow.tsx` — Renders a single server as a table row.
- `ServerStat.tsx` — Renders CPU/Mem/Network sparkline charts using `useServerMetrics` hook.
- `ServerPortsStat.tsx` — Renders `usedPorts/totalPorts`.
- `ServerSSHStat.tsx` — Subscribes to `CONNECT_SERVER_SUBSCRIPTION` for SSH status.
- `ServerTrafficStat.tsx` — Renders upload/download traffic.
- `useServerItem.ts` — Hook managing SSH connected state and edit handler.

Also read the GraphQL queries at `src/queries/server.ts`.

- [ ] **Step 2: Rewrite `ServerList.tsx`**

The new ServerList should be a single table-in-card. Key structure:

```tsx
// Imports: React, translation, Apollo, UI components
// Keep: GET_SERVERS_QUERY, SERVER_METRIC_SUBSCRIPTION, metricsMap state
// Keep: ServerInfoModal with local state (infoModalOpen, selectedServerId)
// Remove: framer-motion, ServerCard, ServerRow, listStyle toggle
// Remove: useServerItem import

<PageHeader title={t("Servers")} description={t("Manage remote servers...")}>
  <Button onClick={handleAdd}><Plus /> {t("Add Server")}</Button>
</PageHeader>

{/* Use gap-5 / mt-5 between page sections — the design system spacing convention */}
<Card className="mt-5">
  {loading ? (
    <CardContent>
      {/* Skeleton table rows */}
    </CardContent>
  ) : !servers?.length ? (
    <CardContent>
      <EmptyState title={t("No servers yet")} message={t("Add a server to get started.")}>
        <Button onClick={handleAdd}>{t("Add Server")}</Button>
      </EmptyState>
    </CardContent>
  ) : (
    <>
      <CardContent>
        <Table>
          <TableHeader>
            <TableRow>
              <TableHead>Server</TableHead>
              <TableHead>SSH</TableHead>
              <TableHead>Ports</TableHead>
              <TableHead>Traffic</TableHead>
              <TableHead>CPU</TableHead>
              <TableHead>Mem</TableHead>
              <TableHead>Disk</TableHead>
              <TableHead className="text-right">Actions</TableHead>
            </TableRow>
          </TableHeader>
          <TableBody>
            {servers.map((server) => {
              const metric = metricsMap.get(server.id);
              return (
                <TableRow key={server.id}>
                  {/* Server: name + address */}
                  <TableCell>
                    <div className="font-medium">{server.name || server.address}</div>
                    <div className="text-xs text-muted-foreground">{server.address}</div>
                  </TableCell>
                  {/* SSH: colored dot badge */}
                  <TableCell>
                    <Badge variant={sshStatus === "connected" ? "secondary" : "outline"}>
                      <span className={cn("size-1.5 rounded-full", colorClass)} />
                      {label}
                    </Badge>
                  </TableCell>
                  {/* Ports — GraphQL fields are portUsed and portTotal */}
                  <TableCell className="text-sm text-muted-foreground">
                    {server.portUsed ?? "—"} / {server.portTotal ?? "—"}
                  </TableCell>
                  {/* Traffic */}
                  <TableCell>
                    <div className="text-xs">↑ {formatBytes(metric?.upload)}</div>
                    <div className="text-xs text-muted-foreground">↓ {formatBytes(metric?.download)}</div>
                  </TableCell>
                  {/* CPU */}
                  <TableCell>
                    <div className="flex items-center gap-2">
                      <Progress value={metric?.cpu ?? 0} className="w-16 h-1.5" />
                      <span className="text-xs text-muted-foreground w-8">{metric?.cpu ?? 0}%</span>
                    </div>
                  </TableCell>
                  {/* Mem — same pattern */}
                  {/* Disk — same pattern */}
                  {/* Actions */}
                  <TableCell className="text-right">
                    <Button variant="ghost" size="sm" onClick={() => handleEdit(server.id)}>Edit</Button>
                    <DropdownMenu>...</DropdownMenu>
                  </TableCell>
                </TableRow>
              );
            })}
          </TableBody>
        </Table>
      </CardContent>
      <CardFooter className="flex items-center justify-between border-t pt-4">
        <span className="text-sm text-muted-foreground">{count} {t("servers")}</span>
        <Paginator ... />
      </CardFooter>
    </>
  )}
</Card>

<ServerInfoModal open={infoModalOpen} onOpenChange={setInfoModalOpen} serverId={selectedServerId} onSuccess={handleModalSuccess} />
```

**Data sources to preserve from old code:**
- `useQuery(GET_SERVERS_QUERY, { variables: { limit, offset } })` — paginated server list
- `client.subscribe({ query: SERVER_METRIC_SUBSCRIPTION })` — real-time metrics stored in `metricsMap`
- The `metricsMap` is a `Record<number, MetricData>` (keyed by numeric `serverId`) built from subscription events. Preserve the exact type from the old code — do NOT use `Map<string, ...>` as the serverId is a number.
- Ensure `fsRootUsedPct` from the subscription is captured in the MetricData and used for the Disk column's Progress bar.

**SSH status logic:** The old `useServerItem` derives SSH connected state from `lastSeen` timestamp (10 min threshold) and metric `isOnline`. The old `ServerSSHStat` had an active probe via `CONNECT_SERVER_SUBSCRIPTION` — this is intentionally simplified to passive-only in the table view (the active probe/retry can be accessed from the server detail page). Inline the passive derivation as a helper function:

```tsx
function getSshStatus(server: Server, metric?: MetricData): "connected" | "error" | "unknown" {
  if (metric?.isOnline) return "connected";
  if (server.lastSeen) {
    const tenMinAgo = Date.now() - 10 * 60 * 1000;
    if (new Date(server.lastSeen).getTime() > tenMinAgo) return "connected";
  }
  return "unknown";
}
```

**Byte formatting:** Port the existing `readableSize` utility or create a simple inline helper:

```tsx
function formatBytes(bytes?: number): string {
  if (!bytes || bytes === 0) return "0 B";
  const units = ["B", "KB", "MB", "GB", "TB"];
  const i = Math.floor(Math.log(bytes) / Math.log(1024));
  return `${(bytes / Math.pow(1024, i)).toFixed(i > 0 ? 1 : 0)} ${units[i]}`;
}
```

- [ ] **Step 3: Delete old server components**

```bash
cd /home/lei/workspace/created/aurora/frontend
rm src/features/server/ServerCard.tsx
rm src/features/server/ServerRow.tsx
rm src/features/server/ServerStat.tsx
rm src/features/server/ServerPortsStat.tsx
rm src/features/server/ServerTrafficStat.tsx
rm src/features/server/ServerSSHStat.tsx
rm src/hooks/useServerItem.ts
```

- [ ] **Step 4: Type-check**

```bash
cd /home/lei/workspace/created/aurora/frontend && npx tsc --noEmit
```

Expected: Zero errors. If errors appear from other pages importing deleted components, those pages are out of scope — but the deleted components should ONLY be imported by ServerList (confirmed by the explore agent).

- [ ] **Step 5: Commit**

```bash
cd /home/lei/workspace/created/aurora/frontend
git add src/features/server/ServerList.tsx
git rm src/features/server/ServerCard.tsx src/features/server/ServerRow.tsx src/features/server/ServerStat.tsx src/features/server/ServerPortsStat.tsx src/features/server/ServerTrafficStat.tsx src/features/server/ServerSSHStat.tsx src/hooks/useServerItem.ts
git commit -m "feat: redesign Servers page as table-in-card with Progress bars and SSH dot indicators"
```

---

## Task 6: Restyle ServerInfoModal to FormDialog pattern

**Files:**
- Modify: `frontend/src/features/server/ServerInfoModal.tsx`

- [ ] **Step 1: Read the current file**

Read the full `ServerInfoModal.tsx` to understand its form structure, tabs, and field layout.

- [ ] **Step 2: Restyle form fields**

Replace raw `Label` + `Input` pairs with `Field` + `FieldLabel` + `Input` from the radix-nova components:

```diff
- <Label htmlFor="name">Name</Label>
- <Input id="name" ... />
+ <Field>
+   <FieldLabel>Name</FieldLabel>
+   <Input ... />
+ </Field>
```

Wrap groups of fields in `<FieldGroup>`:

```tsx
import { Field, FieldGroup, FieldLabel } from "@/components/ui/field"
```

Use `InputGroup` where appropriate, e.g., for address:port:

```tsx
import { InputGroup, InputGroupInput, InputGroupAddon, InputGroupText } from "@/components/ui/input-group"

<Field>
  <FieldLabel>Address</FieldLabel>
  <InputGroup>
    <InputGroupInput value={address} onChange={...} placeholder="192.168.1.1" />
    <InputGroupAddon align="inline-end">
      <InputGroupText>:{port}</InputGroupText>
    </InputGroupAddon>
  </InputGroup>
</Field>
```

- [ ] **Step 3: Type-check**

```bash
cd /home/lei/workspace/created/aurora/frontend && npx tsc --noEmit
```

- [ ] **Step 4: Commit**

```bash
cd /home/lei/workspace/created/aurora/frontend
git add src/features/server/ServerInfoModal.tsx
git commit -m "style: restyle ServerInfoModal with Field/FieldGroup/InputGroup pattern"
```

---

## Task 7: Redesign FileCenter as table-in-card

**Files:**
- Rewrite: `frontend/src/features/file/FileCenter.tsx`
- Delete: `frontend/src/features/file/FileCard.tsx`
- Delete: `frontend/src/features/file/FileRow.tsx`

- [ ] **Step 1: Read all file feature files**

Read `FileCenter.tsx`, `FileCard.tsx`, `FileRow.tsx`, `FileModal.tsx`, `FilePreviewModal.tsx` to understand the data flow, modals, and actions per file row (preview, download, bindings, delete).

Also read `src/queries/file.ts` for the GraphQL queries.

- [ ] **Step 2: Rewrite `FileCenter.tsx` as table-in-card**

Same pattern as ServerList. Structure:

```tsx
<PageHeader title={t("Files")} description={t("Upload configuration files, binaries, certificates, and secrets to deploy across your servers.")}>
  <Button onClick={() => setFileModalOpen(true)}><Upload /> {t("Upload File")}</Button>
</PageHeader>

<Card className="mt-5">
  {loading ? (
    <CardContent>{/* Skeleton rows */}</CardContent>
  ) : !files?.length ? (
    <CardContent>
      <EmptyState title={t("No files yet")} message={t("Upload a file to get started.")} />
    </CardContent>
  ) : (
    <>
      <CardContent>
        <Table>
          <TableHeader>
            <TableRow>
              <TableHead>Name</TableHead>
              <TableHead>Type</TableHead>
              <TableHead>Size</TableHead>
              <TableHead>Version</TableHead>
              <TableHead>Updated</TableHead>
              <TableHead className="text-right">Actions</TableHead>
            </TableRow>
          </TableHeader>
          <TableBody>
            {files.map((file) => (
              <TableRow key={file.id}>
                <TableCell>
                  <div className="flex items-center gap-2">
                    {/* File type icon */}
                    <span className="font-medium">{file.name}</span>
                  </div>
                </TableCell>
                <TableCell>
                  <Badge variant={file.type === "SECRET" ? "destructive" : "secondary"}>
                    {file.type}
                  </Badge>
                </TableCell>
                <TableCell className="text-sm text-muted-foreground">{formatSize(file.size)}</TableCell>
                <TableCell className="text-sm text-muted-foreground">{file.version}</TableCell>
                <TableCell className="text-sm text-muted-foreground">{formatDate(file.updatedAt)}</TableCell>
                <TableCell className="text-right">
                  {/* Bindings button, Download button, DropdownMenu with Preview/Delete */}
                </TableCell>
              </TableRow>
            ))}
          </TableBody>
        </Table>
      </CardContent>
      <CardFooter className="flex items-center justify-between border-t pt-4">
        <span className="text-sm text-muted-foreground">{count} {t("total")}</span>
        <Paginator ... />
      </CardFooter>
    </>
  )}
</Card>

{/* Modals rendered inline */}
<FileModal open={fileModalOpen} onOpenChange={setFileModalOpen} onSuccess={refetch} />
<FilePreviewModal open={previewOpen} onOpenChange={setPreviewOpen} file={selectedFile} />
```

**Key changes:**
- Remove `FileCard` and `FileRow` — inline the table rows directly
- Fold the per-row actions (preview, download, bindings, delete) into a `DropdownMenu` + dedicated action buttons
- Fold the `BindingModal` state from the old `FileCard`/`FileRow` into `FileCenter`
- The delete confirmation `AlertDialog` moves inline per-row or into the dropdown
- Port formatting helpers from old code (`readableSize`, date formatting)

- [ ] **Step 3: Delete old file components**

```bash
cd /home/lei/workspace/created/aurora/frontend
rm src/features/file/FileCard.tsx
rm src/features/file/FileRow.tsx
```

- [ ] **Step 4: Type-check**

```bash
cd /home/lei/workspace/created/aurora/frontend && npx tsc --noEmit
```

- [ ] **Step 5: Commit**

```bash
cd /home/lei/workspace/created/aurora/frontend
git add src/features/file/FileCenter.tsx
git rm src/features/file/FileCard.tsx src/features/file/FileRow.tsx
git commit -m "feat: redesign FileCenter as table-in-card with inline actions"
```

---

## Task 8: Redesign DeploymentList as table-in-card

**Files:**
- Rewrite: `frontend/src/features/deployment/DeploymentList.tsx`

- [ ] **Step 1: Read the current file**

Read `DeploymentList.tsx`, `DeploymentStatusBadge.tsx`, `DeployModal.tsx`, `DeploymentDetailModal.tsx` to understand the data flow.

Also read `src/queries/deployment.ts` for the GraphQL queries.

- [ ] **Step 2: Rewrite `DeploymentList.tsx` as table-in-card**

Same pattern. Structure:

```tsx
<PageHeader title={t("Deployments")} description={`Server #${serverId}`}>
  <Button variant="outline" onClick={handleBack}><ArrowLeft /> {t("Back")}</Button>
  <Button variant="outline" onClick={() => navigate(portsPath)}>{t("Ports")}</Button>
  <Button variant="outline" onClick={refetch}>{t("Refresh")}</Button>
  <Button onClick={() => setDeployModalOpen(true)}><Rocket /> {t("Deploy")}</Button>
</PageHeader>

<Card className="mt-5">
  {loading ? (
    <CardContent>{/* Skeleton rows */}</CardContent>
  ) : !deployments?.length ? (
    <CardContent>
      <EmptyState title={t("No deployments yet")} message={t("Create a deployment from a bound service to start running workloads on this server.")} />
    </CardContent>
  ) : (
    <>
      <CardContent>
        <Table>
          <TableHeader>
            <TableRow>
              <TableHead>Service</TableHead>
              <TableHead>Status</TableHead>
              <TableHead>Port</TableHead>
              <TableHead>Created</TableHead>
              <TableHead className="text-right">Actions</TableHead>
            </TableRow>
          </TableHeader>
          <TableBody>
            {deployments.map((deployment) => (
              <TableRow key={deployment.id}>
                <TableCell className="font-medium">{deployment.serviceTitle || deployment.bindingTitle}</TableCell>
                <TableCell><DeploymentStatusBadge status={deployment.status} /></TableCell>
                <TableCell className="text-sm text-muted-foreground">{deployment.port?.portNumber ?? "—"}</TableCell>
                <TableCell className="text-sm text-muted-foreground">{formatDate(deployment.createdAt)}</TableCell>
                <TableCell className="text-right">
                  <Button variant="ghost" size="sm" onClick={() => openDetail(deployment)}>Details</Button>
                </TableCell>
              </TableRow>
            ))}
          </TableBody>
        </Table>
      </CardContent>
      <CardFooter className="flex items-center justify-between border-t pt-4">
        <span className="text-sm text-muted-foreground">{count} {t("total")}</span>
        <Paginator ... />
      </CardFooter>
    </>
  )}
</Card>

{/* Modals */}
<DeployModal ... />
<DeploymentDetailModal ... />
```

**Key changes:**
- Replace `PageSection` wrapper with the table-in-card `Card` pattern
- Replace `PageHeader` `onAdd` usage with `children` pattern
- Keep `DeploymentStatusBadge` as-is (already clean)
- Keep the existing modal state management (already uses inline Dialog from the migration)
- Port the Back/Ports/Refresh navigation buttons from the current implementation

- [ ] **Step 3: Type-check**

```bash
cd /home/lei/workspace/created/aurora/frontend && npx tsc --noEmit
```

- [ ] **Step 4: Commit**

```bash
cd /home/lei/workspace/created/aurora/frontend
git add src/features/deployment/DeploymentList.tsx
git commit -m "feat: redesign DeploymentList as table-in-card"
```

---

## Task 9: Visual verification

- [ ] **Step 1: Ensure docker stack is running**

```bash
cd /home/lei/workspace/created/aurora && docker compose restart frontend
```

Wait for Vite to start.

- [ ] **Step 2: Browse and screenshot key pages**

Using agent-browser or manual browser:

1. Navigate to `http://aurora.localhost:8060/app/servers` — verify table-in-card layout, Progress bars, SSH dots, bg-muted background
2. Navigate to `http://aurora.localhost:8060/app/files` — verify table-in-card layout, file type badges, action buttons
3. Click into a server → verify deployment table-in-card, status badges, Deploy button
4. Open the "Add Server" dialog — verify Field/FieldGroup form pattern
5. Check that the sidebar and navbar are unaffected

- [ ] **Step 3: Fix any visual issues found**

Address spacing, alignment, or rendering problems found during verification.

- [ ] **Step 4: Final commit if fixes were needed**

```bash
cd /home/lei/workspace/created/aurora/frontend
git add -A src/
git commit -m "fix: visual polish from design verification pass"
```
