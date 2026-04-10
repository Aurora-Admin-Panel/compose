# Frontend Migration: DaisyUI to shadcn/ui + TypeScript

**Date:** 2026-03-24
**Branch:** `feature/migrate-frontend-to-shadcn`
**Status:** Approved

## Overview

Full migration of the Aurora frontend from DaisyUI 5 + JavaScript to shadcn/ui + TypeScript. Includes React 18 to 19 upgrade, Redux removal (consolidate to Jotai + Apollo), and service editor refactor.

## Decisions

| Decision | Choice | Rationale |
|---|---|---|
| Migration strategy | Build shadcn component library first, then incremental in-place feature migration (bulk swap only if DaisyUI config must break) | Keeps app working throughout, components designed holistically |
| Language | TypeScript (strict) | Files converted per-feature during migration |
| React version | 19.x | Aligned with shadcn-template |
| State management | Jotai only (remove Redux) | Eliminate dual-system complexity |
| Theming | Multi-theme via CSS classes + shadcn variables | 4 initial themes, extensible |
| Font | Noto Sans Variable | From shadcn-template |
| `tailwind-mix.js` plugin | Keep available, but prefer shadcn CSS variables for new code | No rework needed, backward compatible |
| Service editor | Refactor during migration (break up 1114-line monolith) | Overdue, natural opportunity |
| Framer Motion | Keep alongside `tw-animate-css` | Different purposes: Framer for orchestrated transitions, tw-animate for CSS utility animations |
| `react-hook-form` | Keep as-is | Already has good TS support, used by service editor's `useDynamicForm` |
| Vite React plugin | Keep `@vitejs/plugin-react-swc` (SWC) | Already in use, faster builds, SWC supports React 19 |

## Section 1: Foundation Setup

### TypeScript

- Add `tsconfig.json` with strict mode, `@/*` path aliases (based on shadcn-template config)
- Add `tsconfig.app.json` (ES2022 target, ESNext module) and `tsconfig.node.json` (ES2023, for vite config)
- Add `vite-env.d.ts` and type declarations for non-TS dependencies
- Rename entry points: `main.jsx` -> `main.tsx`, `App.jsx` -> `App.tsx`
- Remaining files converted from `.jsx` -> `.tsx` as each feature is migrated

### React 19

- Bump `react` and `react-dom` from 18.2 to 19.x
- Replace `react-helmet` with `react-helmet-async` (react-helmet uses legacy string refs, incompatible with React 19)
- Audit `react-loading` for React 19 compatibility; replace if needed
- Audit remaining deps for deprecated patterns (legacy context, string refs)

### Tailwind/CSS Coexistence Layer

DaisyUI and shadcn CSS variables can coexist during migration while DaisyUI remains in the tree:
- DaisyUI uses `--color-base-100`, `--color-primary`, etc. with `data-theme` attribute
- shadcn uses `--background`, `--foreground`, `--primary`, etc. with CSS classes

If token conflicts make coexistence awkward, it is acceptable to let DaisyUI styling degrade or break during the migration window rather than adding complex compatibility shims. DaisyUI is temporary and will be removed in Phase 5.

Changes to `index.css`:
- Keep DaisyUI plugin active during migration
- Add shadcn imports: `shadcn/tailwind.css`, `tw-animate-css`
- Add shadcn `@theme inline` block with CSS variable mappings
- Add theme class definitions (see Section 2 for details)
- Add `@fontsource-variable/noto-sans` import
- Preserve custom breakpoints (`ssm: 320px`, `xs: 475px`) in shadcn `@theme` block (used by `ServerPorts`, `ServerUsers`)

New files:
- `src/lib/utils.ts` — `cn()` utility (clsx + tailwind-merge)
- `components.json` — shadcn CLI configuration

### Package Changes

**Add:**
- `radix-ui` — headless UI primitives
- `class-variance-authority` — component variant system
- `clsx`, `tailwind-merge` — class composition
- `shadcn` — CLI for adding components
- `tw-animate-css` — animation utilities
- `@fontsource-variable/noto-sans` — font
- `typescript`, `@types/react`, `@types/react-dom` — TypeScript tooling
- `react-helmet-async` — React 19-compatible helmet

**Remove (Phase 5, after full migration):**
- `daisyui`
- `@reduxjs/toolkit`, `redux-persist`, `react-redux`
- `classnames` (replaced by `cn()` / `clsx`)
- `react-helmet` (replaced by `react-helmet-async`)

### `classnames` to `cn()` Migration

The `classnames` package is imported in ~30 files. Replace with `cn()` (clsx + tailwind-merge) as each feature is migrated:
- When converting a file from JSX to TSX, replace `import classnames from 'classnames'` with `import { cn } from '@/lib/utils'`
- `cn()` is a drop-in replacement for `classnames()` with the added benefit of Tailwind class deduplication
- Remove `classnames` package in Phase 5 after all files are converted

## Section 2: Theme System

### Architecture

Each theme is a named CSS class (e.g., `.theme-aurora-classic`, `.theme-sunset`) applied to `<html>`. Each class sets all shadcn CSS variables using OKLCH values.

**Dark mode variant:** The shadcn-template's `@custom-variant dark (&:is(.dark *))` must be reworked. Instead of a single `.dark` class, dark themes apply both their named class AND `.dark`:
```html
<!-- Light theme -->
<html class="theme-aurora-classic">
<!-- Dark theme -->
<html class="theme-sunset dark">
```
This ensures shadcn's `dark:` variant works correctly for all dark themes. The theme provider determines `color-scheme` per theme (light or dark) and applies `.dark` accordingly.

**CSS structure:**
```css
/* Default/fallback (aurora-classic) */
:root { --background: oklch(...); --foreground: oklch(...); ... }

/* Each theme overrides all variables */
.theme-aurora-classic { --background: ...; --primary: ...; ... }
.theme-sunset { --background: ...; --primary: ...; ... }
.theme-morning { --background: ...; --primary: ...; ... }
.theme-midnight { --background: ...; --primary: ...; ... }
```

### Initial Themes (4)

| Theme | Type | Primary | Base |
|---|---|---|---|
| `aurora-classic` | light | Purple (#7E3AF2) | White |
| `sunset` | dark | Coral (#EE8679) | Dark navy |
| `morning` | light | Terracotta (#D26A5D) | Light |
| `midnight` | dark | Purple | Dark (new) |

Each theme defines the full shadcn variable set: `--background`, `--foreground`, `--primary`, `--primary-foreground`, `--secondary`, `--secondary-foreground`, `--muted`, `--muted-foreground`, `--accent`, `--accent-foreground`, `--destructive`, `--border`, `--input`, `--ring`, `--card`, `--card-foreground`, `--popover`, `--popover-foreground`, `--sidebar-*`, `--chart-*`, `--radius`.

### Theme Provider

- Custom `ThemeProvider` (rewritten from shadcn-template to support named themes, not just light/dark toggle)
- Keep Jotai `themeSelectionAtom` for localStorage persistence
- "Auto" mode: `prefers-color-scheme` -> aurora-classic (light) or sunset (dark)
- Apply theme class + `.dark` (for dark themes) to `<html>` element
- Keyboard shortcut `d` for quick light/dark toggle (cycles between last-used light and dark themes)

### DaisyUI Theme Reference

Existing 3 custom DaisyUI theme definitions preserved in `src/styles/daisyui-themes-reference.css` (not imported, reference only).

### Theme Switcher UI

Migrate `ThemeSwitch` / `ThemeMenuItems` to TSX with shadcn `DropdownMenu`. Shows 4 themes + "Auto" option.

## Section 3: shadcn Component Library

### Components to Build

| shadcn Component | Replaces | Notes |
|---|---|---|
| `Button` | `btn btn-primary btn-ghost btn-sm` | Extend with Aurora sizes |
| `Input` | `input input-bordered input-error` | With error state styling |
| `Textarea` | `textarea textarea-bordered` | |
| `Select` | `select select-bordered` | Radix Select |
| `Checkbox` | `checkbox` | |
| `Toggle` | `toggle` | Radix Switch |
| `Badge` | `badge badge-primary` | |
| `Card` | Custom `rounded-xl border` cards | Standardize pattern |
| `Dialog` | `modal modal-box modal-backdrop` | Radix Dialog, replaces ModalShell |
| `Sheet` | `drawer drawer-side` | Radix Sheet for mobile sidebar |
| `DropdownMenu` | `dropdown` + `<details>` wrapper | Radix DropdownMenu |
| `Tabs` | `tabs tab tab-active` | |
| `Alert` | `alert alert-info alert-success` | |
| `Progress` | `progress` | |
| `Tooltip` | (new) | |
| `Label` | (new) | Form labels |
| `Separator` | (new) | Replaces `divider` |
| `ScrollArea` | (new) | Scrollable panels |
| `Skeleton` | (new) | Loading states |
| `Table` | Raw `<table>` with Tailwind | Standardized |

**Not needed / deferred:** NavigationMenu, Accordion, Calendar/DatePicker, Command/Combobox.

### Component Location

`src/components/ui/` — matches shadcn convention and `components.json` aliases.

### Approach

Use `npx shadcn@latest add <component>` for the base, then customize styling to match Aurora's design language. Components needing significant customization (e.g., Dialog integrating with modal atom system) are built on top of the shadcn base.

### Custom Layout Components

- `AppShell` — replaces DaisyUI drawer layout (sidebar + content area)
- `NavBar` — migrated from DaisyUI navbar to Tailwind + shadcn primitives
- `SideBar` — migrated from DaisyUI menu to custom shadcn-styled sidebar

## Section 4: Feature Migration Order

Incremental migration. Each feature converts JSX -> TSX and DaisyUI -> shadcn. If DaisyUI config must break at any point, remaining features are bulk-swapped.

### Phase 1 — Core Shell + Shared Infrastructure

1. Convert core atoms to TypeScript: `atoms/auth.ts`, `atoms/modal.ts`, `atoms/theme.ts`, `atoms/notification.ts`, `atoms/layout.ts` (consumed by nearly every feature, must be done first)
2. Migrate notification system from Redux to Jotai notification atom (cross-cutting concern, blocks clean migration of all features that do error handling)
3. Rework GraphQL codegen for TypeScript/Apollo migration:
   - Point schema to `http://aurora.localhost:8060/api/graphql` (or equivalent local backend endpoint when needed)
   - Update document globs to include `.ts` and `.tsx`, not only `.jsx`
   - Move generated base types from `store/apis/types.generated.ts` to `src/types/generated.ts`
   - Remove RTK Query-oriented generation from the migration path; missing GraphQL coverage is documented as TODO, not a blocker
4. `Layout.tsx` + `features/layout/NavBar.tsx` + `features/layout/SideBar.tsx` — the app skeleton
5. `features/modal/ModalManager.tsx` — rewrite with shadcn Dialog, keep Jotai modal atom stack
6. `features/theme/` — new theme switcher with shadcn DropdownMenu
7. `features/i18n/` — language switcher
8. `features/Notification.tsx` — shared notification component
9. `features/Paginator.tsx` — shared pagination component

### Phase 2 — Auth + Simple Features

10. `features/auth/` — login, create account (Input, Button, Card)
11. `features/about/` — static page
12. `features/user/` — user management (Table, Badge, Dialog)

### Phase 3 — Core Business Features

13. `features/server/` — server list, cards, stats
14. `features/port/` — port management modals
15. `features/file/` — file management

### Phase 4 — Complex Features

16. `features/deployment/` — DeployModal, DeploymentList, BindingModal
17. `features/service-editor/` — refactor + migrate (see Section 6)

### Phase 5 — Cleanup

18. Remove DaisyUI plugin + package
19. Remove Redux + redux-persist + react-redux
20. Remove legacy `store/` directory
21. Remove `classnames` package
22. Remove `react-helmet` package
23. Clean up remaining `.jsx` files, dead imports, unused CSS
24. Review `tailwind-safelist.js` — remove if dynamic class safelist is no longer needed, or convert to TS
25. Remove DaisyUI theme reference if no longer needed

### Shared Infrastructure

Converted as each consuming feature is migrated:
- `hooks/` — convert to `.ts`/`.tsx`
- `queries/` — add types
- `graphql.js` -> `graphql.ts`
- `routes.js` -> `routes.ts`
- `i18n.js` -> `i18n.ts`

### `src/apis/` Directory

The `src/apis/` directory (5 files: auth, ports, servers, users, utils) uses `axios` for REST calls. `apis/auth.js` is used by the Jotai auth reducer for token validation. These are kept and converted to TypeScript during migration. Whether to replace them with Apollo/GraphQL calls is out of scope — document as a future TODO if applicable.

## Section 5: Redux Removal

### Strategy

Per-feature during Phases 2-4:
1. Audit what the feature pulls from Redux
2. For RTK Query endpoints: confirm Apollo covers the same data, remove RTK Query usage
3. For persisted Redux state: migrate to Jotai `atomWithStorage`
4. If a feature relies on Redux for data not yet available via GraphQL: document what did not migrate cleanly as a TODO for later GraphQL implementation (do not block migration)

### Cross-Cutting Concerns (Phase 1)

- **Notification system:** `showNotification` Redux thunk is used across features for error handling. Migrate to Jotai `notificationManager` atom early in Phase 1 to unblock all subsequent feature migrations.
- **WebSocket manager:** `store/websocketManager.ts` and related Redux websocket plumbing are legacy and should be removed. The real app uses GraphQL subscriptions via Apollo, so websocket manager removal is part of the Redux cleanup, not a separate investigation.
- **Generated types:** `store/apis/types.generated.ts` exports `FileTypeEnum` used by `FileModal` and `ServerInfoModal`. Move to `src/types/generated.ts` in Phase 1.

### Final Cleanup (Phase 5)

- Remove `Provider` + `PersistGate` wrappers from `main.tsx`
- Delete `store/` directory
- Remove packages: `@reduxjs/toolkit`, `redux-persist`, `react-redux`

## Section 6: Service Editor Refactor

### ParamEditorPanel Split

The 1114-line `ParamEditorPanel.jsx` is broken into focused modules:

| New File | Responsibility |
|---|---|
| `ParamEditorPanel.tsx` | Shell — renders selected param's editor, manages selection state |
| `ParamList.tsx` | Left sidebar list of params with add/remove/reorder |
| `EmitConfigEditor.tsx` | Emit target configuration (arg, flag, env, file, stdin, pos) |
| `ValidationEditor.tsx` | Validation rules (min, max, pattern, etc.) |
| `ConditionEditor.tsx` | Conditional visibility rules |
| `UIConfigEditor.tsx` | UI hints (grid, placeholder, description) |
| `ParamTypeEditor.tsx` | Type selection + type-specific options |

Each module receives the current param draft and an `onChange` callback. The shell orchestrates them in tabs or sections.

### Field Components Migration

- Swap DaisyUI form classes for shadcn `Input`, `Select`, `Checkbox`, `Textarea`, `Label`
- `FieldShell` becomes a wrapper using shadcn `Label` + error display
- `ListField` and `ObjectField` keep recursive rendering logic, restyled with shadcn components
- `react-hook-form` integration stays as-is (already well-typed)
