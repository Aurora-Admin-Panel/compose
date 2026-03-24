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

## Section 1: Foundation Setup

### TypeScript

- Add `tsconfig.json` with strict mode, `@/*` path aliases (based on shadcn-template config)
- Add `tsconfig.app.json` (ES2022 target, ESNext module) and `tsconfig.node.json` (ES2023, for vite config)
- Add `vite-env.d.ts` and type declarations for non-TS dependencies
- Rename entry points: `main.jsx` -> `main.tsx`, `App.jsx` -> `App.tsx`
- Remaining files converted from `.jsx` -> `.tsx` as each feature is migrated

### React 19

- Bump `react` and `react-dom` from 18.2 to 19.x
- Audit for deprecated patterns (legacy context, string refs)

### Tailwind/CSS Coexistence Layer

DaisyUI and shadcn CSS variables can coexist during migration:
- DaisyUI uses `--color-base-100`, `--color-primary`, etc. with `data-theme` attribute
- shadcn uses `--background`, `--foreground`, `--primary`, etc. with CSS classes

**One conflict:** `--border` (DaisyUI = border width `1px`, shadcn = border color). Resolution: rename DaisyUI's `--border: 1px` to `--border-width: 1px` during coexistence.

Changes to `index.css`:
- Keep DaisyUI plugin active during migration
- Add shadcn imports: `shadcn/tailwind.css`, `tw-animate-css`
- Add shadcn `@theme inline` block with CSS variable mappings
- Add theme class definitions (`:root` / `.dark` and custom themes)
- Add `@fontsource-variable/noto-sans` import

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

**Remove (Phase 5, after full migration):**
- `daisyui`
- `@reduxjs/toolkit`, `redux-persist`, `react-redux`

## Section 2: Theme System

### Architecture

Each theme is a CSS class (e.g., `.theme-aurora-classic`) applied to `<html>`. Each class sets all shadcn CSS variables (`--background`, `--foreground`, `--primary`, etc.) using OKLCH values.

### Initial Themes (4)

| Theme | Type | Primary | Base |
|---|---|---|---|
| `aurora-classic` | light | Purple (#7E3AF2) | White |
| `sunset` | dark | Coral (#EE8679) | Dark navy |
| `morning` | light | Terracotta (#D26A5D) | Light |
| `midnight` | dark | Purple | Dark (new) |

Each theme defines the full shadcn variable set: `--background`, `--foreground`, `--primary`, `--primary-foreground`, `--secondary`, `--secondary-foreground`, `--muted`, `--muted-foreground`, `--accent`, `--accent-foreground`, `--destructive`, `--border`, `--input`, `--ring`, `--card`, `--card-foreground`, `--popover`, `--popover-foreground`, `--sidebar-*`, `--chart-*`, `--radius`.

### Theme Provider

- New `ThemeProvider` component (similar to shadcn-template's context-based approach)
- Keep Jotai `themeSelectionAtom` for localStorage persistence
- "Auto" mode: `prefers-color-scheme` -> aurora-classic (light) or sunset (dark)
- Apply theme class to `<html>` element
- Keyboard shortcut `d` for quick light/dark toggle

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

### Phase 1 — Core Shell

1. `Layout.tsx` + `features/layout/NavBar.tsx` + `features/layout/SideBar.tsx`
2. `features/modal/ModalManager.tsx` — rewrite with shadcn Dialog, keep Jotai modal atom stack
3. `features/theme/` — new theme switcher with shadcn DropdownMenu
4. `features/i18n/` — language switcher

### Phase 2 — Auth + Simple Features

5. `features/auth/` — login, create account (Input, Button, Card)
6. `features/about/` — static page
7. `features/user/` — user management (Table, Badge, Dialog)

### Phase 3 — Core Business Features

8. `features/server/` — server list, cards, stats
9. `features/port/` — port management modals
10. `features/file/` — file management

### Phase 4 — Complex Features

11. `features/deployment/` — DeployModal, DeploymentList, BindingModal
12. `features/service-editor/` — refactor + migrate (see Section 6)

### Phase 5 — Cleanup

13. Remove DaisyUI plugin + package
14. Remove Redux + redux-persist + react-redux
15. Remove legacy `store/` directory
16. Clean up remaining `.jsx` files, dead imports, unused CSS
17. Remove DaisyUI theme reference if no longer needed

### Shared Infrastructure

Converted as each consuming feature is migrated:
- `atoms/` — add TypeScript types
- `hooks/` — convert to `.ts`/`.tsx`
- `queries/` — add types
- `graphql.js` -> `graphql.ts`
- `routes.js` -> `routes.ts`
- `i18n.js` -> `i18n.ts`

## Section 5: Redux Removal

### Strategy

Per-feature during Phases 2-4:
1. Audit what the feature pulls from Redux
2. For RTK Query endpoints: confirm Apollo covers the same data, remove RTK Query usage
3. For persisted Redux state: migrate to Jotai `atomWithStorage`
4. If a feature relies on Redux for data not yet available via GraphQL: document as a TODO for later GraphQL implementation (do not block migration)

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
