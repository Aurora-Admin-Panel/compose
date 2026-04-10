# Design System Foundation — Spec

## Summary

Establish the radix-nova design language across the Aurora frontend: global styles (font, background, typography), shared page patterns (TableCard, FormDialog, EmptyState, etc.), layout tweaks, and a full redesign of the Servers page as proof-of-concept. Subsequent sub-projects will redesign remaining pages using these patterns.

## Decisions

| Decision | Choice |
|---|---|
| Font | Noto Sans Variable via `@fontsource-variable/noto-sans` |
| Page background | `bg-muted` (light gray) with white cards on top |
| Density | Slightly more spacious than demo (`gap-5`, `p-5 md:p-6`) |
| Servers layout | Table-in-card (single Card wrapping a Table) |
| Forms | `Field` / `FieldGroup` / `InputGroup` everywhere |
| Proof page | Servers (most complex, exercises all patterns) |

## 1. Global Styles

### Font

Install the font package: `npm install @fontsource-variable/noto-sans`

In `index.css`, add:

```css
@import "@fontsource-variable/noto-sans";
```

In the `@theme inline` block, set:

```css
--font-sans: 'Noto Sans Variable', sans-serif;
```

Remove the existing `--font-sans: var(--font-sans);` self-reference. No separate heading font — everything uses `--font-sans`. Note: `--font-heading: var(--font-sans);` also exists in the `@theme inline` block and will correctly inherit the new Noto Sans font — leave it as-is.

### Page background

The `SidebarInset` content area in `Layout.tsx` gets `bg-muted`. Cards (white) sit on top of the gray surface, creating visual depth.

### Typography scale

| Element | Current | New |
|---|---|---|
| Page titles | `text-2xl font-extrabold` | `text-lg font-semibold` |
| Section descriptions | varies | `text-sm text-muted-foreground` |
| Table body text | `text-sm` | `text-sm` (unchanged) |
| Card titles | `text-base font-medium` | `text-base font-medium` (unchanged) |

### Spacing

| Context | Demo | Aurora (slightly more spacious) |
|---|---|---|
| Card grid gap | `gap-4` | `gap-5` |
| Container padding | `p-4 sm:p-6` | `p-5 md:p-6` |
| Card inner padding | component default (`p-6`) | component default (unchanged) |
| Container max-width | none | `max-w-7xl` |

## 2. Shared Page Patterns

### PageHeader

Simplified page header. Not wrapped in a Card — sits directly on the `bg-muted` surface.

```tsx
<div className="flex items-center justify-between">
  <div>
    <h1 className="text-lg font-semibold">{title}</h1>
    {description && (
      <p className="text-sm text-muted-foreground">{description}</p>
    )}
  </div>
  <div className="flex items-center gap-2">
    {actions}
  </div>
</div>
```

Update `features/ui/PageHeader.tsx` to match this structure. Remove any Card wrapping, large font sizes, or extrabold weights.

### TableCard pattern

Not a new component — a composition pattern used by every page that displays tabular data:

```tsx
<Card>
  <CardHeader>
    <div className="flex items-center justify-between">
      <div>
        <CardTitle>{title}</CardTitle>
        <CardDescription>{description}</CardDescription>
      </div>
      {headerActions}
    </div>
  </CardHeader>
  <CardContent>
    <Table>...</Table>
  </CardContent>
  <CardFooter className="flex items-center justify-between border-t pt-4">
    <span className="text-sm text-muted-foreground">{count} total</span>
    <Paginator ... />
  </CardFooter>
</Card>
```

Pages implement this pattern inline — no wrapper component needed.

### EmptyState

Replace the current custom `EmptyState` component with the zip's `Empty` component:

```tsx
import { Empty, EmptyHeader, EmptyTitle, EmptyDescription, EmptyContent } from "@/components/ui/empty"

<Empty>
  <EmptyHeader>
    <EmptyTitle>No servers yet</EmptyTitle>
    <EmptyDescription>Add a server to get started.</EmptyDescription>
  </EmptyHeader>
  <EmptyContent>
    <Button>Add Server</Button>
  </EmptyContent>
</Empty>
```

The `Empty` component exports: `Empty`, `EmptyHeader`, `EmptyTitle`, `EmptyDescription`, `EmptyContent`, `EmptyMedia`. There is no `EmptyActions` — use `EmptyContent` for action buttons.

This goes inside the `CardContent` area of a TableCard when there's no data.

### FormDialog pattern

All modals with forms use `Field` / `FieldGroup` from the zip:

```tsx
<Dialog open={open} onOpenChange={onOpenChange}>
  <DialogContent>
    <DialogHeader>
      <DialogTitle>{title}</DialogTitle>
      <DialogDescription>{description}</DialogDescription>
    </DialogHeader>
    <FieldGroup>
      <Field>
        <FieldLabel>Name</FieldLabel>
        <Input ... />
      </Field>
      <Field>
        <FieldLabel>Address</FieldLabel>
        <InputGroup>
          <InputGroupInput placeholder="192.168.1.1" />
          <InputGroupAddon align="inline-end">
            <InputGroupText>:22</InputGroupText>
          </InputGroupAddon>
        </InputGroup>
      </Field>
    </FieldGroup>
    <DialogFooter>
      <Button variant="outline" onClick={() => onOpenChange(false)}>Cancel</Button>
      <Button type="submit">Save</Button>
    </DialogFooter>
  </DialogContent>
</Dialog>
```

Replace raw `Label` + `Input` + `div` stacking with this pattern in all modals.

### DataLoading / ThemedSuspense

- `DataLoading.tsx` already uses `Skeleton` — keep it as-is for the 13 consumers outside scope
- The Servers page rewrite uses `Skeleton` directly for table-shaped loading states (rows of skeleton cells)
- `ThemedSuspense.tsx`: update to centered `Spinner` on `bg-muted`

### Paginator

Keep the current `Paginator` component as-is — it already uses valid radix-nova sizes (`size="icon-sm"` for nav buttons, `size="sm"` for Select). The TableCard pattern places it in `CardFooter`.

## 3. Servers Page Redesign

### Current state

The servers page has:
- `ServerList.tsx` — parent component with table/card toggle
- `ServerRow.tsx` — table row per server
- `ServerCard.tsx` — card per server (grid mode)
- `ServerStat.tsx`, `ServerPortsStat.tsx`, `ServerTrafficStat.tsx`, `ServerSSHStat.tsx` — stat sub-components
- `ServerInfoModal.tsx` — add/edit server dialog
- `chart/Chart.tsx`, `chart/Sparkline.tsx` — chart components

### New structure

**ServerList.tsx** — Rewrite as a single table-in-card:

```
PageHeader: "Servers" / "Manage remote servers..."    [+ Add Server]

Card
├── CardContent
│   └── Table
│       ├── TableHeader: Server | SSH | Ports | Traffic | CPU | Mem | Disk | Actions
│       └── TableBody: rows...
└── CardFooter: "{n} servers" + Paginator
```

**Table columns:**

| Column | Content |
|---|---|
| Server | Name (`font-medium`) + address below (`text-xs text-muted-foreground`) |
| SSH | Colored dot: green=connected, red=error, gray=unknown. Use `Badge` with dot indicator |
| Ports | `{used}/{total}` text |
| Traffic | `↑ {upload}` / `↓ {download}` compact format with `text-xs` |
| CPU | `Progress` bar (`w-16 h-1.5`) + `{value}%` text in `text-muted-foreground` |
| Mem | Same as CPU |
| Disk | Same as CPU |
| Actions | `Button` "Edit" + `DropdownMenu` trigger (three dots) for "View" (navigate to server detail), Delete |

**Empty state:** `Empty` component inside `CardContent` when no servers.

**Remove:**
- `ServerCard.tsx` — No longer needed (table-only view)
- `ServerRow.tsx` — Replaced by inline table rows in `ServerList.tsx`
- `ServerStat.tsx`, `ServerPortsStat.tsx`, `ServerTrafficStat.tsx`, `ServerSSHStat.tsx` — Stats are now inline Progress bars in table cells
- `framer-motion` animations in `ServerList.tsx` — The staggered card animations are removed with the card grid

**Keep:**
- `ServerContainer.tsx` — Needed as a React Router route wrapper for nested routes (`/app/servers/:serverId`). Fix the React Router v5 `matchPath` API call to use v6 syntax: `matchPath("/app/servers", location.pathname)` instead of `matchPath({ path: "/app/servers", exact: true }, ...)`.
- `chart/Sparkline.tsx` — May be used in future stat cards, keep but not used on this page initially
- `chart/Chart.tsx` — Same
- `ServerInfoModal.tsx` — Restyle to FormDialog pattern

### ServerInfoModal redesign

Convert to FormDialog pattern:
- Replace raw `Label` + `Input` pairs with `Field` + `FieldLabel` + `Input`
- Use `FieldGroup` to wrap the form fields
- Use `InputGroup` where appropriate (e.g., address:port input)
- Keep the existing tabs if present (General, SSH config, etc.)
- Keep the AlertDialog for delete confirmation

## 4. Layout Tweaks

### SidebarInset content area

In `Layout.tsx`, add `bg-muted` to the content wrapper:

```diff
- <div className="flex-1 overflow-auto">
+ <div className="flex-1 overflow-auto bg-muted">
```

### Container

Add `max-w-7xl` to prevent content stretching on ultrawide monitors:

```diff
- <div className="container p-4 md:p-6">
+ <div className="container max-w-7xl p-5 md:p-6">
```

### Sidebar and navbar

No changes. They already use the correct styling from the migration.

## Files Changed

### New/reinstalled packages
- `@fontsource-variable/noto-sans`

### Modified files
- `frontend/src/index.css` — Font import + `--font-sans` variable
- `frontend/src/Layout.tsx` — `bg-muted` on content area, `max-w-7xl` container
- `frontend/src/features/ui/PageHeader.tsx` — Simplified to flex row, smaller typography
- `frontend/src/features/ui/EmptyState.tsx` — Rewrite to use `Empty` component
- `frontend/src/features/ThemedSuspense.tsx` — Centered `Spinner` on `bg-muted`
- `frontend/src/features/server/ServerList.tsx` — Full rewrite as table-in-card
- `frontend/src/features/server/ServerContainer.tsx` — Fix React Router v6 `matchPath` call
- `frontend/src/features/server/ServerInfoModal.tsx` — Restyle to FormDialog pattern

### Deleted files
- `frontend/src/features/server/ServerCard.tsx`
- `frontend/src/features/server/ServerRow.tsx`
- `frontend/src/features/server/ServerStat.tsx`
- `frontend/src/features/server/ServerPortsStat.tsx`
- `frontend/src/features/server/ServerTrafficStat.tsx`
- `frontend/src/features/server/ServerSSHStat.tsx`
- `frontend/src/hooks/useServerItem.ts` — Logic folded into ServerList.tsx

### Untouched
- All other pages (auth, files, services, ports, users, deployments, about)
- All GraphQL queries
- All atoms except layout-related
- `frontend/src/hooks/useServerMetrics.ts` — Not needed for the new table (which uses snapshot metrics from subscription), but kept for future server detail views
- `frontend/src/features/ui/PageSection.tsx` — Still used by other pages (files, deployments, services, ports). Will be deprecated when those pages are redesigned.
- `frontend/src/features/DataLoading.tsx` — Already uses Skeleton. Keep as-is; imported by 13 files outside scope. Servers page uses Skeleton directly for table loading state.
- `frontend/src/features/server/ServerContainer.tsx` — Route wrapper, kept with v6 matchPath fix
- Backend
- Sidebar and navbar

## Out of Scope

- Redesigning auth pages (Login, CreateAccount)
- Redesigning files page
- Redesigning services/service editor pages
- Redesigning ports/users pages
- Redesigning deployment pages
- Redesigning about/themes pages
- Dark mode adjustments (zinc dark palette is already set)

These are subsequent sub-projects that will use the patterns established here.
