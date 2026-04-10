# Frontend Migration: DaisyUI to shadcn/ui + TypeScript

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Migrate the Aurora frontend from DaisyUI 5 + JavaScript to shadcn/ui + TypeScript, upgrade React 18→19, remove Redux in favor of Jotai, and refactor the service editor.

**Architecture:** Build shadcn component library on top of Radix UI primitives with CVA variants. Multi-theme system using CSS classes on `<html>` that set shadcn CSS variables. Incremental in-place feature migration while DaisyUI coexists — DaisyUI degradation is acceptable during the migration window.

**Tech Stack:** React 19, TypeScript (strict), Vite 7 + SWC, Tailwind CSS 4, shadcn/ui (Radix + CVA), Jotai, Apollo Client, Framer Motion, i18next

**Spec:** `docs/superpowers/specs/2026-03-24-shadcn-migration-design.md`

**Known codebase quirks:**
- `frontend/src/features/auth/CreateAccoount.jsx` has a typo (double "o") — rename to `CreateAccount.tsx` during migration
- `frontend/src/polyfills/symbolObservable.js` is imported in `main.jsx` — was needed for old Redux; remove when Redux is removed
- `frontend/src/plugins/tailwind-mix.js` is imported in `index.css` — keep as-is per spec (available but not preferred for new code)
- `react-loading` is used in `DataLoading.jsx`, `ThemedSuspense.jsx`, `ServerSSHStat.jsx` — uses `createReactClass`, likely incompatible with React 19; replace with shadcn `Skeleton` or CSS spinner during React 19 upgrade
- `phosphor-react` is in `package.json` but unused — remove in Phase 5
- `frontend/codegen.yml` exists and is configured for RTK Query (`typescript-rtk-query` plugin) — must be updated in Task 9
- `postcss.config.cjs` and `tailwind.config.cjs` are vestigial with Tailwind v4 CSS config — review in Phase 5

**Scope note for feature tasks (Phases 2-4):** When a task says "All files in `frontend/src/features/X/`", this means **all files including subdirectories**. The Files section lists key files but is not exhaustive — convert every `.jsx`/`.js` file in the directory.

---

## Foundation

### Task 1: TypeScript + Package Foundation

**Files:**
- Modify: `frontend/package.json`
- Replace: `frontend/tsconfig.json` (existing file has incompatible structure — overwrite entirely)
- Create: `frontend/tsconfig.app.json`
- Create: `frontend/tsconfig.node.json`
- Create: `frontend/src/vite-env.d.ts`
- Modify: `frontend/vite.config.js` → `frontend/vite.config.ts`
- Delete: `frontend/jsconfig.json`

- [ ] **Step 1: Install new dependencies**

```bash
cd frontend
npm install @fontsource-variable/noto-sans class-variance-authority clsx tailwind-merge radix-ui shadcn tw-animate-css react-helmet-async
npm install -D typescript @types/react @types/react-dom @types/node
```

- [ ] **Step 2: Upgrade React to 19**

```bash
cd frontend
npm install react@^19 react-dom@^19
npm install -D @types/react@^19 @types/react-dom@^19
```

Check for peer dependency conflicts. If `react-helmet` fails on peer deps, use `--legacy-peer-deps`. It will be removed in Task 6.

**Important:** `react-loading` uses `createReactClass` which is incompatible with React 19. Before proceeding, replace it:

1. Uninstall: `npm uninstall react-loading`
2. Find consumers: `grep -r "react-loading" src/ -l` (expect: `DataLoading.jsx`, `ThemedSuspense.jsx`, `ServerSSHStat.jsx`)
3. Replace each with a simple CSS spinner or shadcn Skeleton. For `ThemedSuspense.jsx` (used as Suspense fallback in `main.jsx`):

```tsx
export default function ThemedSuspense() {
  return (
    <div className="flex h-full items-center justify-center">
      <div className="size-8 animate-spin rounded-full border-4 border-muted border-t-primary" />
    </div>
  );
}
```

Apply same pattern for `DataLoading.jsx`. `ServerSSHStat.jsx` can use shadcn `Skeleton` once available (Task 4), or the CSS spinner for now.

- [ ] **Step 3: Create tsconfig files**

`frontend/tsconfig.json` — project references root:
```json
{
  "files": [],
  "references": [
    { "path": "./tsconfig.app.json" },
    { "path": "./tsconfig.node.json" }
  ],
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  }
}
```

`frontend/tsconfig.app.json`:
```json
{
  "compilerOptions": {
    "tsBuildInfoFile": "./node_modules/.tmp/tsconfig.app.tsbuildinfo",
    "target": "ES2022",
    "useDefineForClassFields": true,
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "types": ["vite/client"],
    "skipLibCheck": true,
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "verbatimModuleSyntax": false,
    "moduleDetection": "force",
    "noEmit": true,
    "jsx": "react-jsx",
    "strict": true,
    "noUnusedLocals": false,
    "noUnusedParameters": false,
    "noFallthroughCasesInSwitch": true,
    "allowJs": true,
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["src"]
}
```

Note: `allowJs: true` enables incremental migration. `noUnusedLocals` and `noUnusedParameters` set to `false` during migration to avoid noise from partially-converted files. Tighten after Phase 5.

`frontend/tsconfig.node.json`:
```json
{
  "compilerOptions": {
    "tsBuildInfoFile": "./node_modules/.tmp/tsconfig.node.tsbuildinfo",
    "target": "ES2023",
    "lib": ["ES2023"],
    "module": "ESNext",
    "types": ["node"],
    "skipLibCheck": true,
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "verbatimModuleSyntax": false,
    "moduleDetection": "force",
    "noEmit": true,
    "strict": true,
    "noFallthroughCasesInSwitch": true
  },
  "include": ["vite.config.ts"]
}
```

- [ ] **Step 4: Create vite-env.d.ts**

`frontend/src/vite-env.d.ts`:
```typescript
/// <reference types="vite/client" />
```

- [ ] **Step 5: Rename vite.config.js → vite.config.ts**

Convert `frontend/vite.config.js` to TypeScript. Keep `@vitejs/plugin-react-swc` (not the Babel plugin from shadcn-template). Keep the dev proxy config.

```typescript
import path from "path"
import tailwindcss from "@tailwindcss/vite"
import react from "@vitejs/plugin-react-swc"
import svgr from "vite-plugin-svgr"
import { defineConfig } from "vite"

export default defineConfig({
  plugins: [tailwindcss(), svgr(), react()],
  resolve: {
    alias: {
      "@": path.resolve(__dirname, "./src"),
    },
  },
  server: {
    proxy: {
      "/api": {
        target: "http://localhost:8000",
        changeOrigin: true,
      },
    },
  },
})
```

- [ ] **Step 6: Delete jsconfig.json**

Remove `frontend/jsconfig.json` — superseded by tsconfig.json.

- [ ] **Step 7: Add components.json for shadcn CLI**

`frontend/components.json`:
```json
{
  "$schema": "https://ui.shadcn.com/schema.json",
  "style": "radix-maia",
  "rsc": false,
  "tsx": true,
  "tailwind": {
    "config": "",
    "css": "src/index.css",
    "baseColor": "olive",
    "cssVariables": true,
    "prefix": ""
  },
  "iconLibrary": "lucide",
  "rtl": false,
  "aliases": {
    "components": "@/components",
    "utils": "@/lib/utils",
    "ui": "@/components/ui",
    "lib": "@/lib",
    "hooks": "@/hooks"
  }
}
```

- [ ] **Step 8: Verify the app still builds**

```bash
cd frontend && npm run build
```

Fix any TypeScript or dependency errors. The app should still work with existing JSX files since `allowJs: true`.

- [ ] **Step 9: Commit**

```bash
git add frontend/
git commit -m "feat: add TypeScript config and shadcn dependencies"
```

---

### Task 2: CSS Foundation + Theme Variables

**Files:**
- Modify: `frontend/src/index.css`
- Create: `frontend/src/lib/utils.ts`
- Create: `frontend/src/styles/daisyui-themes-reference.css`

- [ ] **Step 1: Create cn() utility**

`frontend/src/lib/utils.ts`:
```typescript
import { clsx, type ClassValue } from "clsx"
import { twMerge } from "tailwind-merge"

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs))
}
```

- [ ] **Step 2: Copy existing DaisyUI theme definitions to reference file**

Copy the 3 custom `@plugin 'daisyui/theme'` blocks from `frontend/src/index.css` into `frontend/src/styles/daisyui-themes-reference.css` (not imported, reference only).

- [ ] **Step 3: Fix --border variable conflict**

In the DaisyUI theme blocks in `frontend/src/index.css`, rename `--border: 1px` to `--border-width: 1px` in all three custom themes (aurora-classic, sunset, morning). This is a **required** step — DaisyUI uses `--border` for border width, but shadcn uses it for border color. They will conflict.

- [ ] **Step 4: Add shadcn CSS layer to index.css**

Add these imports after the existing DaisyUI imports in `frontend/src/index.css`:

```css
@import "tw-animate-css";
@import "shadcn/tailwind.css";
@import "@fontsource-variable/noto-sans";
```

Add the `@custom-variant` directive:
```css
@custom-variant dark (&:is(.dark *));
```

Add the `@theme inline` block with shadcn variable mappings (copy from `shadcn-template/src/index.css` lines 8-49).

Add custom breakpoints inside the `@theme inline` block:
```css
  --breakpoint-ssm: 320px;
  --breakpoint-xs: 475px;
```

- [ ] **Step 5: Add Aurora theme definitions**

Add 4 theme classes after the `@theme inline` block. Each sets all shadcn CSS variables.

`:root` block = aurora-classic (light, default fallback).

`.theme-aurora-classic` = same values as `:root`.

`.theme-sunset.dark` = dark theme with coral primary. Derive OKLCH values from existing DaisyUI sunset theme colors (#151726 base, #EE8679 primary).

`.theme-morning` = light theme with terracotta primary. Derive from existing DaisyUI morning theme (#FDFDFE base, #D26A5D primary).

`.theme-midnight.dark` = new dark theme with purple primary. Design purple-toned dark palette.

Each theme must define: `--background`, `--foreground`, `--card`, `--card-foreground`, `--popover`, `--popover-foreground`, `--primary`, `--primary-foreground`, `--secondary`, `--secondary-foreground`, `--muted`, `--muted-foreground`, `--accent`, `--accent-foreground`, `--destructive`, `--border`, `--input`, `--ring`, `--chart-1` through `--chart-5`, `--radius`, `--sidebar` and all `--sidebar-*` variants.

- [ ] **Step 6: Add base layer styles**

```css
@layer base {
  * {
    @apply border-border outline-ring/50;
  }
  body {
    @apply bg-background text-foreground;
  }
  html {
    font-family: var(--font-sans);
  }
}
```

- [ ] **Step 7: Verify build still works**

```bash
cd frontend && npm run build
```

DaisyUI and shadcn CSS should coexist. DaisyUI styling degradation is acceptable.

- [ ] **Step 8: Commit**

```bash
git add frontend/src/
git commit -m "feat: add shadcn CSS foundation and Aurora theme definitions"
```

---

### Task 3: shadcn Component Library — Form Primitives

**Files:**
- Create: `frontend/src/components/ui/button.tsx`
- Create: `frontend/src/components/ui/input.tsx`
- Create: `frontend/src/components/ui/textarea.tsx`
- Create: `frontend/src/components/ui/select.tsx`
- Create: `frontend/src/components/ui/checkbox.tsx`
- Create: `frontend/src/components/ui/switch.tsx`
- Create: `frontend/src/components/ui/label.tsx`

- [ ] **Step 1: Add components via shadcn CLI**

```bash
cd frontend
npx shadcn@latest add button input textarea select checkbox switch label
```

This generates components in `src/components/ui/`. Each uses Radix primitives + CVA + `cn()`.

- [ ] **Step 2: Verify button matches shadcn-template**

Compare generated `button.tsx` with `shadcn-template/src/components/ui/button.tsx`. The shadcn-template version has Aurora-specific customizations (rounded-4xl, icon sizes). If the generated version differs significantly, replace with the template version.

- [ ] **Step 3: Verify all components render**

Create a temporary test: import each component in `App.jsx` (or a scratch file), render them, and confirm no build errors.

```bash
cd frontend && npm run build
```

- [ ] **Step 4: Commit**

```bash
git add frontend/src/components/
git commit -m "feat: add shadcn form primitive components"
```

---

### Task 4: shadcn Component Library — Feedback + Layout

**Files:**
- Create: `frontend/src/components/ui/badge.tsx`
- Create: `frontend/src/components/ui/alert.tsx`
- Create: `frontend/src/components/ui/progress.tsx`
- Create: `frontend/src/components/ui/skeleton.tsx`
- Create: `frontend/src/components/ui/tooltip.tsx`
- Create: `frontend/src/components/ui/card.tsx`
- Create: `frontend/src/components/ui/separator.tsx`
- Create: `frontend/src/components/ui/scroll-area.tsx`
- Create: `frontend/src/components/ui/table.tsx`
- Create: `frontend/src/components/ui/tabs.tsx`

- [ ] **Step 1: Add components via shadcn CLI**

```bash
cd frontend
npx shadcn@latest add badge alert progress skeleton tooltip card separator scroll-area table tabs
```

- [ ] **Step 2: Build check**

```bash
cd frontend && npm run build
```

- [ ] **Step 3: Commit**

```bash
git add frontend/src/components/
git commit -m "feat: add shadcn feedback and layout components"
```

---

### Task 5: shadcn Component Library — Overlay Components

**Files:**
- Create: `frontend/src/components/ui/dialog.tsx`
- Create: `frontend/src/components/ui/sheet.tsx`
- Create: `frontend/src/components/ui/dropdown-menu.tsx`

- [ ] **Step 1: Add components via shadcn CLI**

```bash
cd frontend
npx shadcn@latest add dialog sheet dropdown-menu
```

- [ ] **Step 2: Build check**

```bash
cd frontend && npm run build
```

- [ ] **Step 3: Commit**

```bash
git add frontend/src/components/
git commit -m "feat: add shadcn overlay components (dialog, sheet, dropdown-menu)"
```

---

### Task 6: React 19 Compatibility Fixes

**Files:**
- Modify: `frontend/package.json`
- Modify: files importing `react-helmet`

- [ ] **Step 1: Replace react-helmet with react-helmet-async**

`react-helmet-async` was already installed in Task 1. Remove `react-helmet`:

```bash
cd frontend && npm uninstall react-helmet
```

- [ ] **Step 2: Update imports**

Find all files importing `react-helmet`:

```bash
cd frontend && grep -r "react-helmet" src/ --include="*.jsx" --include="*.tsx" -l
```

In each file, replace:
```javascript
// Before
import { Helmet } from 'react-helmet';
// After
import { Helmet } from 'react-helmet-async';
```

In `main.jsx` (or `main.tsx` after conversion), wrap the app with `HelmetProvider`:
```jsx
import { HelmetProvider } from 'react-helmet-async';
// Wrap around the app tree
<HelmetProvider>
  <App />
</HelmetProvider>
```

- [ ] **Step 3: Audit react-loading**

```bash
cd frontend && grep -r "react-loading" src/ --include="*.jsx" --include="*.tsx" -l
```

Check if `react-loading` works with React 19. If it fails, replace with a simple CSS spinner or shadcn `Skeleton` component.

- [ ] **Step 4: Build + verify**

```bash
cd frontend && npm run build
```

- [ ] **Step 5: Commit**

```bash
git add frontend/
git commit -m "feat: replace react-helmet with react-helmet-async for React 19"
```

---

## Phase 1 — Core Shell + Shared Infrastructure

### Task 7: Convert Core Atoms to TypeScript

**Files:**
- Rename: `frontend/src/atoms/auth.js` → `auth.ts`
- Rename: `frontend/src/atoms/theme.js` → `theme.ts`
- Rename: `frontend/src/atoms/layout.js` → `layout.ts`
- Rename: `frontend/src/atoms/notification.js` → `notification.ts`
- Rename: `frontend/src/atoms/notificationManager.js` → `notificationManager.ts`
- Rename: `frontend/src/atoms/modal.js` → `modal.ts`
- Rename: `frontend/src/atoms/server/limit.js` → `limit.ts` (if exists)

- [ ] **Step 1: Convert auth.ts**

Rename file. Add types for auth state and actions:

```typescript
interface AuthState {
  token: string | null;
  permissions: {
    is_superuser?: boolean;
    is_ops?: boolean;
    user_id?: number;
  };
}

type AuthAction =
  | { type: 'login'; token: string }
  | { type: 'logout' };
```

Type the `authAtom` (atomWithStorage) and `useAuthReducer` hook. Keep the JWT decode logic.

- [ ] **Step 2: Convert remaining atoms**

For each atom file:
1. Rename `.js` → `.ts`
2. Add explicit types for state shape
3. Add types for action/reducer parameters
4. Type the exported hooks

`theme.ts` is trivial (4 lines). `layout.ts` is trivial (3 lines). `notification.ts` needs types for notification objects. `modal.ts` needs types for the modal stack entries and the `useModal()` hook return type.

- [ ] **Step 3: Fix import paths in consumers**

Since we renamed `.js` → `.ts`, imports like `from '@/atoms/auth'` should still resolve (bundler resolves without extension). Verify no explicit `.js` extensions in imports.

- [ ] **Step 4: Build check**

```bash
cd frontend && npm run build
```

- [ ] **Step 5: Commit**

```bash
git add frontend/src/atoms/
git commit -m "feat: convert core atoms to TypeScript"
```

---

### Task 8: Migrate Notification System from Redux to Jotai

**Files:**
- Modify: `frontend/src/atoms/notification.ts`
- Modify: `frontend/src/graphql.js` (uses notification for error handling)
- Modify: any file importing `showNotification` from Redux
- Reference: `frontend/src/store/reducers/notification.js` (to understand API)

- [ ] **Step 1: Audit Redux notification consumers**

```bash
cd frontend && grep -r "showNotification\|notification.*dispatch\|store/reducers/notification" src/ --include="*.jsx" --include="*.tsx" --include="*.js" --include="*.ts" -l
```

Document each consumer and what it does.

- [ ] **Step 2: Ensure Jotai notification atom has equivalent API**

The existing `atoms/notification.ts` already has `notify()` function and `useNotificationsReducer()`. Verify it supports the same operations as the Redux version:
- `addNotification({title, body, type, duration})`
- `removeNotification(id)`
- Auto-dismiss with timeout

If `notify()` is a standalone function (not requiring React context), it can replace `dispatch(showNotification(...))` anywhere.

- [ ] **Step 3: Replace Redux notification imports**

In each consumer file, replace:
```javascript
// Before
import { showNotification } from '@/store/reducers/notification';
dispatch(showNotification({ title, body, type }));

// After
import { notify } from '@/atoms/notification';
notify({ title, body, type });
```

Pay special attention to `graphql.js` error handler — it uses notification for GraphQL errors.

Also check `store/reducers/utils.js` (`handleError`) — this is the centralized error handler. Replace its Redux dispatch with Jotai notify.

- [ ] **Step 4: Build check**

```bash
cd frontend && npm run build
```

- [ ] **Step 5: Commit**

```bash
git add frontend/src/
git commit -m "feat: migrate notification system from Redux to Jotai"
```

---

### Task 9: Relocate Generated Types + GraphQL Codegen

**Files:**
- Create: `frontend/src/types/generated.ts`
- Modify: files importing from `store/apis/types.generated.ts`
- Modify: GraphQL codegen config (if exists)

- [ ] **Step 1: Read codegen config**

The codegen config is at `frontend/codegen.yml`. Read it to understand current plugins and paths.

- [ ] **Step 2: Copy types to new location**

```bash
mkdir -p frontend/src/types
cp frontend/src/store/apis/types.generated.ts frontend/src/types/generated.ts
```

- [ ] **Step 3: Update imports**

Find all consumers:
```bash
cd frontend && grep -r "store/apis/types" src/ -l
```

Replace import paths:
```typescript
// Before
import { FileTypeEnum } from '../../store/apis/types.generated';
// After
import { FileTypeEnum } from '@/types/generated';
```

- [ ] **Step 4: Update codegen.yml**

In `frontend/codegen.yml`:
- Change output path from `src/store/apis/types.generated.ts` to `src/types/generated.ts`
- Update `documents` glob from `'src/**/*.jsx'` to `'src/**/*.{jsx,tsx,ts}'`
- Remove `typescript-rtk-query` plugin (RTK Query codegen)
- Remove `importBaseApiFrom` reference to `src/store/graphqlBaseApi`
- Keep the base `typescript` and `typescript-operations` plugins

Also update any `queries/*.js` files that import from the old `store/apis/types.generated.ts` path to use `@/types/generated`.

- [ ] **Step 5: Build check + commit**

```bash
cd frontend && npm run build
git add frontend/src/types/ frontend/src/
git commit -m "feat: relocate generated GraphQL types to src/types/"
```

---

### Task 10: Theme Provider + Theme System

**Files:**
- Create: `frontend/src/components/theme-provider.tsx`
- Modify: `frontend/src/atoms/theme.ts`
- Modify: `frontend/src/contexts/ThemeContext.jsx` (to be replaced)
- Modify: `frontend/src/main.jsx`

- [ ] **Step 1: Define theme configuration**

Add to `frontend/src/atoms/theme.ts`:

```typescript
export interface ThemeConfig {
  name: string;
  label: string;
  colorScheme: 'light' | 'dark';
}

export const THEMES: ThemeConfig[] = [
  { name: 'aurora-classic', label: 'Aurora Classic', colorScheme: 'light' },
  { name: 'morning', label: 'Morning', colorScheme: 'light' },
  { name: 'sunset', label: 'Sunset', colorScheme: 'dark' },
  { name: 'midnight', label: 'Midnight', colorScheme: 'dark' },
];

export const DEFAULT_LIGHT_THEME = 'aurora-classic';
export const DEFAULT_DARK_THEME = 'sunset';
```

- [ ] **Step 2: Create new ThemeProvider**

`frontend/src/components/theme-provider.tsx`:

Based on `shadcn-template/src/components/theme-provider.tsx` but extended for named themes:
- Theme type: `'auto' | 'aurora-classic' | 'morning' | 'sunset' | 'midnight'`
- Instead of toggling `light`/`dark` classes, apply `.theme-{name}` class + `.dark` for dark themes
- `auto` resolves via `prefers-color-scheme` → `aurora-classic` or `sunset`
- Keep keyboard `d` shortcut (toggle between last-used light and dark theme)
- Keep `disableTransitionsTemporarily()` from template
- Persist to localStorage via Jotai `themeSelectionAtom`

Key logic for applying theme:
```typescript
const config = THEMES.find(t => t.name === resolvedTheme);
root.className = ''; // clear all theme classes
root.classList.add(`theme-${resolvedTheme}`);
if (config?.colorScheme === 'dark') {
  root.classList.add('dark');
}
```

- [ ] **Step 3: Replace ThemeContext in main.jsx**

Replace `import { ThemeProvider } from '@/contexts/ThemeContext'` with the new provider. Remove old `ThemeContext.jsx` file.

- [ ] **Step 4: Delete old theme files**

- Delete `frontend/src/contexts/ThemeContext.jsx`
- Delete `frontend/src/utils/themes.js` (generated DaisyUI theme list — no longer needed)
- Delete `frontend/scripts/generate-themes.mjs` (DaisyUI theme generator)

- [ ] **Step 5: Build check + commit**

```bash
cd frontend && npm run build
git add frontend/src/
git commit -m "feat: implement multi-theme system with shadcn CSS variables"
```

---

### Task 11: Layout Shell Migration

**Files:**
- Rename+Rewrite: `frontend/src/Layout.jsx` → `Layout.tsx`
- Rename+Rewrite: `frontend/src/features/layout/NavBar.jsx` → `NavBar.tsx`
- Rename+Rewrite: `frontend/src/features/layout/SideBar.jsx` → `SideBar.tsx`
- Rename: `frontend/src/routes.js` → `routes.ts`
- Rename+Convert: `frontend/src/features/layout/Error.jsx` → `Error.tsx`
- Rename+Convert: `frontend/src/features/layout/NoMatch.jsx` → `NoMatch.tsx`
- Rename+Convert: `frontend/src/features/layout/Hero.jsx` → `Hero.tsx`
- Delete or Rewrite: `frontend/src/features/layout/Themes.jsx` (10.5K DaisyUI theme demo page — likely obsolete with new theme system; delete if unused, or rewrite as a theme preview page)
- Delete: `frontend/src/features/layout/Footer.jsx` (empty file)

- [ ] **Step 1: Convert routes.ts**

Rename `routes.js` → `routes.ts`. Add type for route config:

```typescript
interface RouteConfig {
  path: string;
  label: string;
  icon: React.ComponentType;
  adminOnly?: boolean;
}
```

- [ ] **Step 2: Rewrite Layout.tsx**

Convert from DaisyUI drawer pattern to a custom AppShell using Tailwind + shadcn `Sheet`:

- Desktop: fixed sidebar (w-60) + content area
- Mobile: `Sheet` (side=left) triggered by hamburger button
- Remove `initializeWebSocket` / `closeWebSocket` imports (legacy Redux websocket)
- Keep token refresh on mount and auth redirect logic
- Replace `classnames` imports with `cn` from `@/lib/utils`

DaisyUI classes to remove: `drawer`, `drawer-toggle`, `drawer-open`, `drawer-content`, `drawer-side`

Replace with Tailwind flex layout:
```tsx
<div className="flex h-screen bg-background">
  {/* Desktop sidebar */}
  <aside className="hidden lg:flex lg:w-60 lg:flex-col border-r border-border">
    <SideBar />
  </aside>
  {/* Mobile sidebar via Sheet */}
  <Sheet open={drawerOpen} onOpenChange={setDrawerOpen}>
    <SheetContent side="left" className="w-60 p-0">
      <SideBar />
    </SheetContent>
  </Sheet>
  {/* Main content */}
  <div className="flex flex-1 flex-col overflow-hidden">
    <NavBar />
    <main className="flex-1 overflow-auto p-4">
      <Outlet />
    </main>
  </div>
</div>
```

- [ ] **Step 3: Rewrite NavBar.tsx**

Replace DaisyUI `navbar` with Tailwind flex layout:
- Sticky top bar with `bg-background border-b border-border`
- Mobile: hamburger button (opens Sheet from Layout)
- Right side: theme switch, language switch, account dropdown (using shadcn `DropdownMenu`)
- Replace DaisyUI avatar dropdown with `DropdownMenu` + `DropdownMenuTrigger` + `DropdownMenuContent`
- Replace `classnames` with `cn`

- [ ] **Step 4: Rewrite SideBar.tsx**

Replace DaisyUI `menu` with custom nav:
- Logo/brand at top
- Nav links from `routes.ts` config
- Active state: `bg-primary/10 text-primary` (keep current style but without DaisyUI classes)
- Use `NavLink` from react-router-dom with `cn()` for conditional active styling
- Replace `classnames` with `cn`

- [ ] **Step 5: Build check + commit**

```bash
cd frontend && npm run build
git add frontend/src/
git commit -m "feat: migrate Layout, NavBar, SideBar to shadcn/Tailwind"
```

---

### Task 12: Modal System Rewrite

**Files:**
- Rename+Rewrite: `frontend/src/features/modal/ModalManager.jsx` → `ModalManager.tsx`
- Rename+Rewrite: `frontend/src/features/ui/ModalShell.jsx` → `ModalShell.tsx`
- Rename+Rewrite: `frontend/src/features/modal/ConfirmationModal.jsx` → `ConfirmationModal.tsx`

- [ ] **Step 1: Rewrite ModalShell.tsx**

Replace DaisyUI `modal-box` with shadcn `Dialog`:

```tsx
import {
  DialogContent,
  DialogHeader,
  DialogTitle,
  DialogFooter,
} from '@/components/ui/dialog';
import { Button } from '@/components/ui/button';

interface ModalShellProps {
  title: string;
  onClose: () => void;
  footer?: React.ReactNode;
  children: React.ReactNode;
  className?: string;
}

export function ModalShell({ title, onClose, footer, children, className }: ModalShellProps) {
  return (
    <>
      <DialogHeader>
        <DialogTitle>{title}</DialogTitle>
      </DialogHeader>
      <div className={cn("py-4", className)}>
        {children}
      </div>
      {footer && <DialogFooter>{footer}</DialogFooter>}
    </>
  );
}
```

- [ ] **Step 2: Rewrite ModalManager.tsx**

Keep the Jotai modal atom stack system. Wrap each modal in shadcn `Dialog`:

```tsx
import { Dialog, DialogContent } from '@/components/ui/dialog';

// For each modal in the stack:
<Dialog
  open={true}
  onOpenChange={(open) => { if (!open) close(modal.id); }}
>
  <DialogContent
    className={cn("sm:max-w-lg", modal.options?.className)}
    style={{ zIndex: 2000 + index * 10 }}
    onPointerDownOutside={(e) => {
      if (!isTop) e.preventDefault();
    }}
  >
    <ModalComponent {...modal.props} modalId={modal.id} close={closeFn} resolve={resolveFn} />
  </DialogContent>
</Dialog>
```

Keep the MODAL_REGISTRY pattern. Keep Esc key handling (Radix Dialog handles this by default).

- [ ] **Step 3: Rewrite ConfirmationModal.tsx**

Type the props and use shadcn `Button`:

```tsx
interface ConfirmationModalProps {
  title?: string;
  message?: string;
  confirmText?: string;
  cancelText?: string;
  resolve: (value: boolean) => void;
  close: () => void;
}
```

- [ ] **Step 4: Build check + commit**

```bash
cd frontend && npm run build
git add frontend/src/features/modal/ frontend/src/features/ui/
git commit -m "feat: rewrite modal system with shadcn Dialog"
```

---

### Task 13: Theme Switcher + Language Switcher Migration

**Files:**
- Rename+Rewrite: `frontend/src/features/theme/ThemeSwitch.jsx` → `ThemeSwitch.tsx`
- Rename+Rewrite: `frontend/src/features/theme/ThemeMenuItems.jsx` → `ThemeMenuItems.tsx`
- Rename+Rewrite: `frontend/src/features/i18n/LanguageSwitch.jsx` → `LanguageSwitch.tsx`
- Rename+Rewrite: `frontend/src/features/i18n/LanguageMenuItems.jsx` → `LanguageMenuItems.tsx`
- Rename: `frontend/src/i18n.js` → `i18n.ts`

- [ ] **Step 1: Rewrite ThemeSwitch + ThemeMenuItems**

Replace DaisyUI `dropdown` with shadcn `DropdownMenu`:

```tsx
import { DropdownMenu, DropdownMenuContent, DropdownMenuItem, DropdownMenuTrigger } from '@/components/ui/dropdown-menu';
import { Button } from '@/components/ui/button';
import { Palette, Check } from 'lucide-react';
import { THEMES } from '@/atoms/theme';
import { useTheme } from '@/components/theme-provider';

export function ThemeSwitch() {
  const { theme, setTheme } = useTheme();
  return (
    <DropdownMenu>
      <DropdownMenuTrigger asChild>
        <Button variant="ghost" size="icon"><Palette className="size-5" /></Button>
      </DropdownMenuTrigger>
      <DropdownMenuContent>
        <DropdownMenuItem onClick={() => setTheme('auto')}>
          Auto {theme === 'auto' && <Check className="ml-auto size-4" />}
        </DropdownMenuItem>
        {THEMES.map(t => (
          <DropdownMenuItem key={t.name} onClick={() => setTheme(t.name)}>
            {t.label} {theme === t.name && <Check className="ml-auto size-4" />}
          </DropdownMenuItem>
        ))}
      </DropdownMenuContent>
    </DropdownMenu>
  );
}
```

- [ ] **Step 2: Rewrite LanguageSwitch + LanguageMenuItems**

Same pattern with `DropdownMenu`. Replace `<details>` dropdown with Radix.

- [ ] **Step 3: Convert i18n.ts**

Rename `i18n.js` → `i18n.ts`. Minimal changes — add type imports if needed.

- [ ] **Step 4: Build check + commit**

```bash
cd frontend && npm run build
git add frontend/src/features/theme/ frontend/src/features/i18n/ frontend/src/i18n.ts
git commit -m "feat: migrate theme and language switchers to shadcn"
```

---

### Task 14: Notification + Paginator + Shared UI Migration

**Files:**
- Rename+Rewrite: `frontend/src/features/Notification.jsx` → `Notification.tsx`
- Rename+Rewrite: `frontend/src/features/Paginator.jsx` → `Paginator.tsx` (if exists)
- Rename+Rewrite: `frontend/src/features/ui/PageHeader.jsx` → `PageHeader.tsx`
- Rename+Rewrite: `frontend/src/features/ui/EmptyState.jsx` → `EmptyState.tsx`
- Rename+Rewrite: `frontend/src/features/ui/dropdown/Dropdown.jsx` → removed (replaced by shadcn DropdownMenu)

- [ ] **Step 1: Rewrite Notification.tsx**

Keep Framer Motion animations. Replace DaisyUI alert classes with shadcn `Alert` or custom Tailwind styling:

DaisyUI `alert-success` → shadcn `bg-green-50 text-green-900 dark:bg-green-950 dark:text-green-100` (or use Alert component with variant).

Keep: fixed positioning, progress bar, click-to-copy, auto-dismiss, pause on hover.

- [ ] **Step 2: Rewrite PageHeader.tsx**

Simple — replace any DaisyUI classes with Tailwind utilities. Add TypeScript props interface.

- [ ] **Step 3: Rewrite EmptyState.tsx**

Simple — type the props, replace DaisyUI classes.

- [ ] **Step 4: Migrate Paginator if it exists**

Replace DaisyUI `btn-group` / pagination classes with shadcn `Button` variants.

- [ ] **Step 5: Delete old Dropdown component**

`features/ui/dropdown/Dropdown.jsx` and `DropdownSubmenu.jsx` are replaced by shadcn `DropdownMenu`. Delete them. Update any remaining imports to use shadcn.

- [ ] **Step 6: Build check + commit**

```bash
cd frontend && npm run build
git add frontend/src/features/
git commit -m "feat: migrate Notification, PageHeader, EmptyState to shadcn"
```

---

### Task 15: Entry Point Migration

**Files:**
- Rename+Rewrite: `frontend/src/main.jsx` → `main.tsx`
- Rename+Rewrite: `frontend/src/App.jsx` → `App.tsx`
- Modify: `frontend/index.html` (update script src)

- [ ] **Step 1: Convert main.tsx**

Rename `main.jsx` → `main.tsx`. Key changes:
- Add `HelmetProvider` wrapper (from react-helmet-async)
- Replace old `ThemeProvider` import with new one from `@/components/theme-provider`
- Keep Redux `Provider` + `PersistGate` for now (still needed by unmigrated features)
- Keep Apollo `ApolloProvider`
- Keep Sentry setup
- Update `index.html` script src from `main.jsx` to `main.tsx`

- [ ] **Step 2: Convert App.tsx**

Rename `App.jsx` → `App.tsx`. Key changes:
- Replace `react-helmet` imports with `react-helmet-async`
- Type the lazy route imports
- Replace `classnames` with `cn` if used
- Keep all route definitions

- [ ] **Step 3: Convert shared infrastructure files**

Rename and add minimal types:
- `graphql.js` → `graphql.ts` — type the Apollo client config
- `routes.js` → `routes.ts` — already done in Task 11 if not

- [ ] **Step 4: Build check + commit**

```bash
cd frontend && npm run build
git add frontend/src/ frontend/index.html
git commit -m "feat: convert entry points (main, App) to TypeScript"
```

---

## Phase 2 — Auth + Simple Features

### Task 16: Auth Feature Migration

**Files:**
- Rename+Rewrite: `frontend/src/features/auth/Login.jsx` → `Login.tsx`
- Rename+Rewrite: `frontend/src/features/auth/CreateAccoount.jsx` → `CreateAccount.tsx` (fix typo in filename)
- Rename+Rewrite: `frontend/src/features/auth/EmailPasswordForm.jsx` → `EmailPasswordForm.tsx`
- Convert: `frontend/src/apis/auth.js` → `auth.ts`

- [ ] **Step 1: Convert apis/auth.ts**

Rename, add types for request/response shapes. Keep axios calls.

- [ ] **Step 2: Migrate auth components**

For each component:
1. Rename `.jsx` → `.tsx`
2. Replace `classnames` with `cn`
3. Replace DaisyUI form classes: `input input-bordered` → shadcn `Input`, `btn btn-primary` → shadcn `Button`
4. Replace DaisyUI `card` with shadcn `Card`
5. Add TypeScript interfaces for props
6. Replace any Redux auth imports with Jotai `useAuthReducer`

- [ ] **Step 3: Build check + commit**

```bash
cd frontend && npm run build
git add frontend/src/features/auth/ frontend/src/apis/
git commit -m "feat: migrate auth feature to shadcn + TypeScript"
```

---

### Task 17: About + User Feature Migration

**Files:**
- Rename+Rewrite: `frontend/src/features/about/About.jsx` → `About.tsx`
- Rename+Rewrite: `frontend/src/features/user/Users.jsx` → `Users.tsx`
- Rename+Rewrite: `frontend/src/features/user/ServerUsers.jsx` → `ServerUsers.tsx`
- Convert: `frontend/src/apis/users.js` → `users.ts`

- [ ] **Step 1: Migrate About page**

Trivial — rename, add types, replace DaisyUI classes with Tailwind/shadcn equivalents.

- [ ] **Step 2: Migrate User components**

Replace DaisyUI `table` classes with shadcn `Table`. Replace `badge` with shadcn `Badge`. Replace `modal` patterns with shadcn `Dialog`.

- [ ] **Step 3: Convert apis/users.ts**

Rename, add request/response types.

- [ ] **Step 4: Build check + commit**

```bash
cd frontend && npm run build
git add frontend/src/features/about/ frontend/src/features/user/ frontend/src/apis/users.ts
git commit -m "feat: migrate About and User features to shadcn + TypeScript"
```

---

## Phase 3 — Core Business Features

### Task 18: Server Feature Migration

**Files:**
- All files in `frontend/src/features/server/` — rename `.jsx` → `.tsx`, rewrite
- Convert: `frontend/src/apis/servers.js` → `servers.ts`
- Convert: `frontend/src/hooks/useServerItem.js` → `useServerItem.ts`
- Convert: `frontend/src/hooks/useServerMetrics.js` → `useServerMetrics.ts`

- [ ] **Step 1: Convert server hooks**

`useServerItem.ts` and `useServerMetrics.ts` — rename, add types for return values and parameters.

- [ ] **Step 2: Migrate server components**

Key replacements:
- `ServerCard` / `ServerRow`: DaisyUI card classes → shadcn `Card`
- `ServerStat` / `ServerPortsStat`: DaisyUI `stats` / `stat` → custom Tailwind or shadcn Card sections
- `ServerInfoModal`: DaisyUI modal → uses ModalShell (already migrated)
- Charts (`Chart.jsx`, `Sparkline.jsx`): Recharts stays, just update wrapper styling
- `classnames` → `cn` in all files
- Any Redux imports → audit and replace or document as TODO

- [ ] **Step 3: Convert apis/servers.ts**

Rename, add request/response types.

- [ ] **Step 4: Build check + commit**

```bash
cd frontend && npm run build
git add frontend/src/features/server/ frontend/src/apis/servers.ts frontend/src/hooks/
git commit -m "feat: migrate server feature to shadcn + TypeScript"
```

---

### Task 19: Port Feature Migration

**Files:**
- All files in `frontend/src/features/port/` — rename `.jsx` → `.tsx`, rewrite
- Convert: `frontend/src/apis/ports.js` → `ports.ts`

- [ ] **Step 1: Migrate port components**

- `PortCard` / `PortSelectCard`: DaisyUI card → shadcn `Card`
- `PortFunctionModal` / `PortRestrictionModal`: already use ModalShell (migrated)
- **Redux dependencies:** `PortRestrictionModal.jsx` uses `useSelector` and `PortCard.jsx` uses `useDispatch`. Replace with Jotai atoms or Apollo queries. If the data isn't available via GraphQL yet, document as TODO.
- Include subdirectories: `function/`, `restriction/` (contains `PortExpiration.jsx`), `PortUsersCard.jsx`, `ServerPorts.jsx`
- Form inputs: DaisyUI `input` / `select` → shadcn `Input` / `Select`
- `classnames` → `cn`

- [ ] **Step 2: Convert apis/ports.ts**

Rename, add types.

- [ ] **Step 3: Build check + commit**

```bash
cd frontend && npm run build
git add frontend/src/features/port/ frontend/src/apis/ports.ts
git commit -m "feat: migrate port feature to shadcn + TypeScript"
```

---

### Task 20: File Feature Migration

**Files:**
- All files in `frontend/src/features/file/` — rename `.jsx` → `.tsx`, rewrite

- [ ] **Step 1: Migrate file components**

- `FileCard` / `FileRow`: DaisyUI styling → Tailwind + shadcn Card
- `FileModal` / `FilePreviewModal`: uses ModalShell (migrated)
- Update `FileTypeEnum` import to `@/types/generated`
- `classnames` → `cn`

- [ ] **Step 2: Build check + commit**

```bash
cd frontend && npm run build
git add frontend/src/features/file/
git commit -m "feat: migrate file feature to shadcn + TypeScript"
```

---

## Phase 4 — Complex Features

### Task 21: Deployment Feature Migration

**Files:**
- All files in `frontend/src/features/deployment/` — rename `.jsx` → `.tsx`, rewrite

- [ ] **Step 1: Migrate deployment components**

- `DeployModal`: Uses `useDynamicForm` + `serviceDefinitionToDynamicSchema` — keep this integration, just restyle the container with shadcn `Dialog`
- `DeploymentList`: DaisyUI table → shadcn `Table`
- `DeploymentDetailModal`: ModalShell (migrated)
- `DeploymentStatusBadge`: DaisyUI badge → shadcn `Badge`
- `BindingModal`: ModalShell + form inputs → shadcn equivalents
- `classnames` → `cn`

- [ ] **Step 2: Build check + commit**

```bash
cd frontend && npm run build
git add frontend/src/features/deployment/
git commit -m "feat: migrate deployment feature to shadcn + TypeScript"
```

---

### Task 22: Service Editor — Field Components Migration

**Files:**
- Rename+Rewrite: `frontend/src/features/service-editor/fields/FieldShell.jsx` → `FieldShell.tsx`
- Rename+Rewrite: `frontend/src/features/service-editor/fields/FieldsRenderer.jsx` → `FieldsRenderer.tsx`
- Rename+Rewrite: `frontend/src/features/service-editor/fields/TextField.jsx` → `TextField.tsx`
- Rename+Rewrite: `frontend/src/features/service-editor/fields/SelectField.jsx` → `SelectField.tsx`
- Rename+Rewrite: `frontend/src/features/service-editor/fields/CheckboxField.jsx` → `CheckboxField.tsx`
- Rename+Rewrite: `frontend/src/features/service-editor/fields/TextAreaField.jsx` → `TextAreaField.tsx`
- Rename+Rewrite: `frontend/src/features/service-editor/fields/ListField.jsx` → `ListField.tsx`
- Rename+Rewrite: `frontend/src/features/service-editor/fields/ObjectField.jsx` → `ObjectField.tsx`
- Rename+Rewrite: `frontend/src/features/service-editor/fields/FieldError.jsx` → `FieldError.tsx`

- [ ] **Step 1: Rewrite FieldShell.tsx**

Replace DaisyUI form-control pattern with shadcn `Label` + error display:

```tsx
import { Label } from '@/components/ui/label';

interface FieldShellProps {
  label?: string;
  error?: string;
  required?: boolean;
  children: React.ReactNode;
  className?: string;
}

export function FieldShell({ label, error, required, children, className }: FieldShellProps) {
  return (
    <div className={cn("space-y-2", className)}>
      {label && (
        <Label>
          {label}
          {required && <span className="text-destructive ml-1">*</span>}
        </Label>
      )}
      {children}
      {error && <p className="text-sm text-destructive">{error}</p>}
    </div>
  );
}
```

- [ ] **Step 2: Migrate individual field components**

For each field:
- `TextField`: DaisyUI `input input-bordered` → shadcn `Input`
- `SelectField`: DaisyUI `select select-bordered` → shadcn `Select`
- `CheckboxField`: DaisyUI `checkbox` → shadcn `Checkbox`
- `TextAreaField`: DaisyUI `textarea textarea-bordered` → shadcn `Textarea`
- `ListField`: keep array logic, restyle add/remove buttons with shadcn `Button`
- `ObjectField`: keep recursive rendering, restyle container

All get TypeScript interfaces. All replace `classnames` with `cn`.

- [ ] **Step 3: Migrate FieldsRenderer.tsx**

Type the schema and field dispatch logic. Keep recursive rendering for nested objects/arrays.

- [ ] **Step 4: Build check + commit**

```bash
cd frontend && npm run build
git add frontend/src/features/service-editor/fields/
git commit -m "feat: migrate service editor field components to shadcn + TypeScript"
```

---

### Task 23: Service Editor — ParamEditorPanel Refactor + Migration

**Files:**
- Rename+Rewrite+Split: `frontend/src/features/service-editor/ParamEditorPanel.jsx` → multiple `.tsx` files
- Create: `frontend/src/features/service-editor/ParamList.tsx`
- Create: `frontend/src/features/service-editor/EmitConfigEditor.tsx`
- Create: `frontend/src/features/service-editor/ValidationEditor.tsx`
- Create: `frontend/src/features/service-editor/ConditionEditor.tsx`
- Create: `frontend/src/features/service-editor/UIConfigEditor.tsx`
- Create: `frontend/src/features/service-editor/ParamTypeEditor.tsx`

- [ ] **Step 1: Read and understand ParamEditorPanel.jsx**

Read the full 1114-line file. Identify the logical sections:
1. Param list (left sidebar with add/remove/reorder)
2. Param type selection + type-specific options
3. Emit configuration (arg, flag, env, file, stdin, pos)
4. Validation rules
5. Conditional visibility
6. UI config hints

- [ ] **Step 2: Extract ParamList.tsx**

Extract the param list sidebar into its own component. Props:
```typescript
interface ParamListProps {
  params: ParamDraft[];
  selectedIndex: number;
  onSelect: (index: number) => void;
  onAdd: () => void;
  onRemove: (index: number) => void;
  onReorder: (fromIndex: number, toIndex: number) => void;
}
```

- [ ] **Step 3: Extract EmitConfigEditor.tsx**

Extract emit target configuration. Props:
```typescript
interface EmitConfigEditorProps {
  emit: EmitConfig;
  paramType: string;
  onChange: (emit: EmitConfig) => void;
}
```

- [ ] **Step 4: Extract ValidationEditor.tsx, ConditionEditor.tsx, UIConfigEditor.tsx, ParamTypeEditor.tsx**

Each gets its own file with typed props and `onChange` callback pattern.

- [ ] **Step 5: Rewrite ParamEditorPanel.tsx as shell**

The shell renders `ParamList` on the left, and the selected param's editors on the right (organized in Tabs using shadcn `Tabs`):
- Tab 1: Type + Basic (ParamTypeEditor)
- Tab 2: Emit (EmitConfigEditor)
- Tab 3: Validation (ValidationEditor)
- Tab 4: Conditions (ConditionEditor)
- Tab 5: UI (UIConfigEditor)

- [ ] **Step 6: Build check + commit**

```bash
cd frontend && npm run build
git add frontend/src/features/service-editor/
git commit -m "feat: refactor and migrate ParamEditorPanel to shadcn + TypeScript"
```

---

### Task 24: Service Editor — Remaining Files Migration

**Files:**
- Rename+Rewrite: `frontend/src/features/service-editor/ServiceEditorPage.jsx` → `ServiceEditorPage.tsx`
- Rename+Rewrite: `frontend/src/features/service-editor/ServiceListPage.jsx` → `ServiceListPage.tsx`
- Rename+Rewrite: `frontend/src/features/service-editor/AuthoringJsonPanel.jsx` → `AuthoringJsonPanel.tsx`
- Rename+Rewrite: `frontend/src/features/service-editor/FormPreviewPanel.jsx` → `FormPreviewPanel.tsx`
- Rename+Rewrite: `frontend/src/features/service-editor/CompileOutputPanel.jsx` → `CompileOutputPanel.tsx`
- Rename+Rewrite: `frontend/src/features/service-editor/useDynamicForm.jsx` → `useDynamicForm.tsx`
- Rename+Rewrite: `frontend/src/features/service-editor/serviceAdapter.js` → `serviceAdapter.ts`

- [ ] **Step 1: Convert serviceAdapter.ts**

Rename, add types for the param→fieldSpec mapping. Key types:
```typescript
interface FieldSpec {
  type: 'text' | 'number' | 'checkbox' | 'select' | 'password' | 'textarea' | 'list' | 'object';
  label?: string;
  required?: boolean;
  default?: unknown;
  options?: { value: string; label: string }[];
  validation?: Record<string, unknown>;
  grid?: Record<string, unknown>;
  children?: Record<string, FieldSpec>;
}
```

- [ ] **Step 2: Convert useDynamicForm.tsx**

Rename, type the hook parameters and return value. Keep `react-hook-form` integration.

- [ ] **Step 3: Migrate panel components**

For each panel:
- Replace DaisyUI classes with Tailwind/shadcn equivalents
- `classnames` → `cn`
- Add TypeScript interfaces

`ServiceEditorPage` orchestrates 3 panels + `ParamEditorPanel` — update imports to new split modules.

- [ ] **Step 4: Migrate ServiceListPage**

Table of service definitions. DaisyUI table → shadcn `Table`. DaisyUI buttons → shadcn `Button`.

- [ ] **Step 5: Build check + commit**

```bash
cd frontend && npm run build
git add frontend/src/features/service-editor/
git commit -m "feat: migrate service editor pages and adapters to shadcn + TypeScript"
```

---

## Phase 5 — Cleanup

### Task 25: Remove DaisyUI + Redux + Dead Code

**Files:**
- Modify: `frontend/package.json`
- Modify: `frontend/src/index.css`
- Modify: `frontend/src/main.tsx`
- Delete: `frontend/src/store/` (entire directory)
- Delete: `frontend/src/features/ui/dropdown/` (replaced by shadcn)
- Delete: `frontend/tailwind.config.cjs` (if no longer needed with Tailwind v4 CSS config)

- [ ] **Step 1: Remove DaisyUI from CSS**

In `frontend/src/index.css`:
- Remove `@plugin 'daisyui'` block
- Remove all `@plugin 'daisyui/theme'` blocks
- Remove DaisyUI-specific custom rules (`.menu` override, `.stat` override)
- Keep shadcn imports, theme definitions, and custom breakpoints

- [ ] **Step 2: Remove Redux from main.tsx**

Remove `Provider`, `PersistGate`, `store`, `persistor` imports and wrappers.

- [ ] **Step 3: Delete store/ directory**

```bash
rm -rf frontend/src/store/
```

- [ ] **Step 4: Uninstall removed packages**

```bash
cd frontend
npm uninstall daisyui @reduxjs/toolkit redux-persist react-redux classnames phosphor-react
```

- [ ] **Step 4b: Remove symbolObservable polyfill**

Delete `frontend/src/polyfills/symbolObservable.js` and remove its import from `main.tsx`. This polyfill was needed for old Redux.

- [ ] **Step 5: Clean up remaining .jsx files**

```bash
cd frontend && find src/ -name "*.jsx" -type f
```

Any remaining `.jsx` files should be renamed to `.tsx` and converted. Also check for `.js` files that should be `.ts`.

- [ ] **Step 6: Review vestigial config files**

- `tailwind-safelist.js`: Check if dynamic Tailwind classes in the safelist are still needed. If DaisyUI classes were the main reason, remove the file. If custom dynamic classes remain, convert to `.ts`.
- `tailwind.config.cjs`: With Tailwind v4 CSS config in `index.css`, this file may be vestigial. Check if anything references it; remove if unused.
- `postcss.config.cjs`: Check if `@tailwindcss/postcss` is still needed or if `@tailwindcss/vite` plugin covers it. Remove if unused.
- `openapi-config.cjs`: REST API codegen config. Out of scope for this migration but note for future cleanup.

- [ ] **Step 7: Clean up unused imports and dead code**

Run the build and fix any warnings about unused imports. Check for any remaining `classnames` imports, `data-theme` references, or DaisyUI class usage.

- [ ] **Step 8: Final build + smoke test**

```bash
cd frontend && npm run build
```

Verify the built app renders correctly with all 4 themes.

- [ ] **Step 9: Remove DaisyUI theme reference (optional)**

If `src/styles/daisyui-themes-reference.css` is no longer useful, delete it.

- [ ] **Step 10: Tighten TypeScript config**

In `tsconfig.app.json`, set:
```json
"noUnusedLocals": true,
"noUnusedParameters": true,
"allowJs": false
```

Fix any resulting errors.

- [ ] **Step 11: Commit**

```bash
git add -A frontend/
git commit -m "feat: remove DaisyUI, Redux, and legacy code — migration complete"
```

- [ ] **Step 12: Convert remaining shared files**

Convert any remaining utility/hook files not yet touched:
- `frontend/src/utils/*.js` → `.ts`
- `frontend/src/hooks/*.js` → `.ts`/`.tsx`
- `frontend/src/apis/utils.js` → `utils.ts`
- `frontend/src/queries/*.js` → `.ts`

```bash
git add frontend/src/
git commit -m "feat: convert remaining shared files to TypeScript"
```
