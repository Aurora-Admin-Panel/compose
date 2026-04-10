# Radix-Nova UI Migration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace all existing shadcn/ui components with radix-nova style, rebuild layout on shadcn Sidebar, replace Jotai modal manager with inline dialogs, switch to Sonner toasts, and migrate all pages.

**Architecture:** Drop-and-replace of 58 radix-nova components, then wave-based page migration. Layout rebuilt with shadcn Sidebar + sticky navbar in SidebarInset. Modal pattern shifts from centralized Jotai atom-based ModalManager to local `useState` + Dialog primitives per page.

**Tech Stack:** React 19, Vite 7, Tailwind CSS 4, shadcn/ui radix-nova style, Radix UI, Sonner, Jotai, Apollo Client, React Router 6

**Spec:** `docs/superpowers/specs/2026-03-24-radix-nova-migration-design.md`

**Zip source:** `/tmp/newui/` (extracted from `/tmp/b_v8Z7GKXAjyF-1774381622427.zip`)

---

## File Map

### New files (from zip)
- `frontend/src/components/ui/*.tsx` — 56 radix-nova components (replacing existing 20)
- `frontend/src/hooks/use-mobile.ts` — mobile detection hook (used by sidebar)

### Major rewrites
- `frontend/src/index.css` — New zinc light/dark theme, stripped custom fonts
- `frontend/src/Layout.tsx` — Rebuilt with shadcn Sidebar + sticky navbar
- `frontend/src/App.tsx` — Remove ModalManager and Notification renders

### Files deleted
- `frontend/src/features/layout/SideBar.tsx`
- `frontend/src/features/layout/NavBar.tsx`
- `frontend/src/features/modal/ModalManager.tsx`
- `frontend/src/features/modal/ConfirmationModal.tsx`
- `frontend/src/features/ui/ModalShell.tsx`
- `frontend/src/features/Notification.tsx`
- `frontend/src/atoms/notification.ts`
- `frontend/src/atoms/notificationManager.ts`
- `frontend/src/atoms/modal.ts`
- `frontend/src/atoms/layout.ts`
- `frontend/src/plugins/tailwind-mix.js`

### Files modified across waves
- All page components in `frontend/src/features/` — updated imports, variant props, modal patterns
- `frontend/src/graphql.ts` — `notify()` → `toast()`
- `frontend/src/main.tsx` — Add Sonner `<Toaster />`
- `frontend/src/features/ui/PageHeader.tsx`, `PageSection.tsx`, `EmptyState.tsx` — new primitives
- `frontend/src/features/DataLoading.tsx`, `Paginator.tsx`, `ThemedSuspense.tsx` — updated

---

## Task 1: Install new dependencies

**Files:**
- Modify: `frontend/package.json`

- [ ] **Step 1: Install new packages**

```bash
cd frontend && npm install sonner vaul embla-carousel-react react-day-picker date-fns input-otp react-resizable-panels
```

- [ ] **Step 2: Verify install succeeded**

```bash
cd frontend && npm ls sonner vaul embla-carousel-react react-day-picker input-otp react-resizable-panels
```

Expected: All packages listed without errors.

- [ ] **Step 3: Commit**

```bash
git add frontend/package.json frontend/package-lock.json
git commit -m "chore: add radix-nova component dependencies (sonner, vaul, embla, etc.)"
```

---

## Task 2: Drop-replace all UI components

**Files:**
- Replace: `frontend/src/components/ui/*.tsx` (20 files replaced, 36 new files added)
- Create: `frontend/src/hooks/use-mobile.ts`

- [ ] **Step 1: Clear existing components and copy radix-nova set**

```bash
# Remove existing UI components
rm frontend/src/components/ui/*.tsx

# Copy all radix-nova components EXCEPT toast.tsx, toaster.tsx, use-mobile.tsx, use-toast.ts
for f in /tmp/newui/components/ui/*.tsx; do
  base=$(basename "$f")
  case "$base" in
    toast.tsx|toaster.tsx|use-mobile.tsx) continue ;;
    *) cp "$f" frontend/src/components/ui/ ;;
  esac
done

# Do NOT copy use-toast.ts from components/ui/
# Copy the hooks version of use-mobile
cp /tmp/newui/hooks/use-mobile.ts frontend/src/hooks/use-mobile.ts
```

- [ ] **Step 2: Strip `"use client"` directives from all copied files**

```bash
cd frontend && sed -i '/^"use client"$/d' src/components/ui/*.tsx
```

- [ ] **Step 3: Verify file count**

```bash
ls frontend/src/components/ui/*.tsx | wc -l
```

Expected: 56 files.

- [ ] **Step 4: Verify imports resolve (no `next-themes` references remain)**

```bash
grep -r "next-themes" frontend/src/components/ui/ frontend/src/hooks/
```

Expected: Only `frontend/src/components/ui/sonner.tsx` should match (to be fixed in next step).

- [ ] **Step 5: Fix sonner.tsx — rewrite next-themes import and theme derivation**

In `frontend/src/components/ui/sonner.tsx`, replace the import and theme logic:

```diff
- import { useTheme } from "next-themes"
+ import { useTheme } from "@/components/theme-provider"
+ import { THEMES } from "@/atoms/theme"
```

Replace the theme destructuring inside the `Toaster` component:

```diff
- const { theme = "system" } = useTheme()
+ const { resolvedTheme } = useTheme()
+ const themeConfig = THEMES.find((t) => t.name === resolvedTheme)
+ const theme = themeConfig?.colorScheme === "dark" ? "dark" : "light"
```

- [ ] **Step 6: Verify no remaining `next-themes` references**

```bash
grep -r "next-themes" frontend/src/
```

Expected: No matches.

- [ ] **Step 7: Commit**

```bash
git add frontend/src/components/ui/ frontend/src/hooks/use-mobile.ts
git commit -m "feat: drop-replace UI components with radix-nova style set"
```

---

## Task 3: Reset CSS theme

**Files:**
- Rewrite: `frontend/src/index.css`
- Delete: `frontend/src/plugins/tailwind-mix.js`

- [ ] **Step 1: Rewrite `index.css`**

Replace the entire file with the new theme. Key changes:
- Use zinc light/dark palette from the zip's `app/globals.css` (at `/tmp/newui/app/globals.css`)
- Add `--destructive-foreground` to both `:root` and `.dark`
- Keep `@import "shadcn/tailwind.css"` and `@import "tw-animate-css"`
- Remove `@fontsource-variable/*` imports
- Remove `@plugin './plugins/tailwind-mix.js'` and `@plugin '@tailwindcss/typography'`
- Remove all `--font-heading` / `--font-sans` custom overrides
- Remove the `h1-h6` rule block that uses `font-family: var(--font-heading)`
- Keep `.theme-*` class scaffolding as empty comments for future use
- Keep `--breakpoint-ssm` and `--breakpoint-xs` if used

The new `index.css` should be structured as:
```css
@import 'tailwindcss';
@import "tw-animate-css";
@import "shadcn/tailwind.css";
@custom-variant dark (&:is(.dark *));

@theme inline {
  /* Use the zip's @theme inline block from /tmp/newui/app/globals.css */
  /* Keep --breakpoint-ssm: 320px and --breakpoint-xs: 475px from current */
  /* Include: --color-destructive-foreground: var(--destructive-foreground); */
}

:root {
  /* Zinc light palette from /tmp/newui/app/globals.css */
  /* Add: --destructive-foreground: oklch(0.985 0 0); (white — for text on destructive backgrounds) */
}

.dark {
  /* Zinc dark palette from /tmp/newui/app/globals.css */
  /* Add: --destructive-foreground: oklch(0.985 0 0); */
}

/* Future themes — empty scaffolding */
/* .theme-aurora-classic { } */
/* .theme-morning { } */
/* .theme-sunset.dark { } */
/* .theme-midnight.dark { } */

@layer base {
  * { @apply border-border outline-ring/50; }
  body { @apply bg-background text-foreground; }
}
```

- [ ] **Step 2: Delete tailwind-mix plugin and empty directory**

```bash
rm frontend/src/plugins/tailwind-mix.js
rmdir frontend/src/plugins/ 2>/dev/null || true
```

- [ ] **Step 3: Global find-and-replace `font-heading` → `font-sans`**

```bash
cd frontend && grep -rl "font-heading" src/ --include="*.tsx" --include="*.ts" | xargs sed -i 's/font-heading/font-sans/g'
```

Note: This is safe because `font-heading` only appears as a Tailwind class name, never as a variable name or string literal (other than in CSS which we just rewrote).

- [ ] **Step 4: Verify no `font-heading` references remain**

```bash
grep -r "font-heading" frontend/src/
```

Expected: No matches.

- [ ] **Step 5: Commit**

```bash
git add frontend/src/index.css frontend/src/plugins/
git add -u frontend/src/  # catch the font-heading replacements
git commit -m "style: reset CSS to zinc light/dark theme, remove custom fonts"
```

---

## Task 4: Migrate notification system to Sonner

**Files:**
- Modify: `frontend/src/graphql.ts:16-33`
- Modify: `frontend/src/features/server/ServerRow.tsx:9` (import), usage lines
- Modify: `frontend/src/features/server/ServerInfoModal.tsx:10` (import), usage lines
- Modify: `frontend/src/features/auth/EmailPasswordForm.tsx:15` (import), usage lines
- Modify: `frontend/src/main.tsx:11` (add Toaster import)
- Modify: `frontend/src/App.tsx:14,46` (remove Notification lazy import and render)
- Delete: `frontend/src/features/Notification.tsx`
- Delete: `frontend/src/atoms/notification.ts`
- Delete: `frontend/src/atoms/notificationManager.ts`

- [ ] **Step 1: Add Sonner `<Toaster />` to `main.tsx`**

In `frontend/src/main.tsx`, add the import and render the Toaster inside the provider stack, after `</HelmetProvider>`:

```diff
+ import { Toaster } from "@/components/ui/sonner"
```

Add `<Toaster />` as a sibling of `<App />` inside `<Suspense>`:

```diff
  <Suspense fallback={<ThemedSuspense />}>
    <App />
+   <Toaster />
  </Suspense>
```

- [ ] **Step 2: Migrate `graphql.ts` — replace `notify()` with `toast()`**

In `frontend/src/graphql.ts`:

```diff
- import { notify } from "./atoms/notification";
+ import { toast } from "sonner";
```

Replace the error handler body (lines 20-33):

```diff
  if (graphQLErrors)
    graphQLErrors.forEach(({ message }) =>
-     notify({
-       title: i18n.t("GraphQL error"),
-       body: message,
-       type: "error",
-     })
+     toast.error(i18n.t("GraphQL error"), { description: message })
    );
  if (networkError)
-   notify({
-     title: i18n.t("Network error"),
-     body: networkError.message,
-     type: "error",
-   })
+   toast.error(i18n.t("Network error"), { description: networkError.message })
```

- [ ] **Step 3: Migrate `ServerInfoModal.tsx` — replace `notify()` with `toast()`**

In `frontend/src/features/server/ServerInfoModal.tsx`:

```diff
- import { notify } from "../../atoms/notification";
+ import { toast } from "sonner";
```

Find all `notify({...})` calls in this file and replace with equivalent `toast()` / `toast.error()` / `toast.success()` calls.

> **Note:** This file will still have compile errors after this step because it also imports `useModal` and `ModalShell` (deleted in Task 5). Those imports are fixed in Task 8 (Wave 2) when the file is converted to inline Dialog. This is expected during the transition period.

- [ ] **Step 4: Migrate `ServerRow.tsx` — replace `useNotificationsReducer` with `toast()`**

In `frontend/src/features/server/ServerRow.tsx`:

```diff
- import { useNotificationsReducer } from "../../atoms/notification";
+ import { toast } from "sonner";
```

Remove the `useNotificationsReducer()` hook call and replace all `addNotification({...})` calls with `toast()` / `toast.error()` / `toast.success()` calls.

- [ ] **Step 5: Migrate `EmailPasswordForm.tsx` — replace `useNotificationsReducer` with `toast()`**

In `frontend/src/features/auth/EmailPasswordForm.tsx`:

```diff
- import { useNotificationsReducer } from "../../atoms/notification";
+ import { toast } from "sonner";
```

Remove the `useNotificationsReducer()` hook call and replace all `addNotification({...})` calls with `toast()` / `toast.error()` / `toast.success()` calls.

- [ ] **Step 6: Remove `<Notification />` from `App.tsx`**

In `frontend/src/App.tsx`:

```diff
- const Notification = lazy(() => import("./features/Notification"));
```

```diff
- <Notification />
```

- [ ] **Step 7: Delete old notification files**

```bash
rm frontend/src/features/Notification.tsx
rm frontend/src/atoms/notification.ts
rm frontend/src/atoms/notificationManager.ts
```

- [ ] **Step 8: Verify no remaining notification atom imports**

```bash
grep -r "atoms/notification" frontend/src/
```

Expected: No matches.

- [ ] **Step 9: Commit**

```bash
git add -A frontend/src/
git commit -m "feat: migrate notification system from Jotai atoms to Sonner toasts"
```

---

## Task 5: Remove modal manager system

**Files:**
- Modify: `frontend/src/App.tsx:13,44` (remove ModalManager lazy import and render)
- Delete: `frontend/src/features/modal/ModalManager.tsx`
- Delete: `frontend/src/features/modal/ConfirmationModal.tsx`
- Delete: `frontend/src/features/ui/ModalShell.tsx`
- Delete: `frontend/src/atoms/modal.ts`

Note: This task only removes the centralized modal infrastructure. Individual modal components (ServerInfoModal, PortFunctionModal, etc.) will be converted to inline Dialog usage in their respective wave tasks. Until then they will be broken — this is acceptable per the spec's "Known Transition Period."

- [ ] **Step 1: Remove `<ModalManager />` from `App.tsx`**

In `frontend/src/App.tsx`:

```diff
- const ModalManager = lazy(() => import("./features/modal/ModalManager"));
```

```diff
- <ModalManager />
```

- [ ] **Step 2: Delete modal infrastructure files and empty directories**

```bash
rm frontend/src/features/modal/ModalManager.tsx
rm frontend/src/features/modal/ConfirmationModal.tsx
rmdir frontend/src/features/modal/
rm frontend/src/features/ui/ModalShell.tsx
rm frontend/src/atoms/modal.ts
```

- [ ] **Step 3: Delete layout atom**

```bash
rm frontend/src/atoms/layout.ts
```

This file only contains `drawerOpenAtom`, which will be replaced by the sidebar's internal state.

- [ ] **Step 4: Commit**

```bash
git add -A frontend/src/
git commit -m "refactor: remove centralized Jotai modal manager and layout atom"
```

---

## Task 6: Rebuild Layout shell with shadcn Sidebar + sticky navbar

**Files:**
- Rewrite: `frontend/src/Layout.tsx`
- Delete: `frontend/src/features/layout/SideBar.tsx`
- Delete: `frontend/src/features/layout/NavBar.tsx`
- Modify: `frontend/src/components/ui/sidebar.tsx` (cookie → localStorage persistence)

- [ ] **Step 1: Fix sidebar.tsx cookie persistence → localStorage**

In `frontend/src/components/ui/sidebar.tsx`, find the cookie-based persistence logic. Replace `document.cookie` reads/writes with `localStorage.getItem("sidebar:state")` / `localStorage.setItem("sidebar:state", ...)`. The sidebar component uses a `SIDEBAR_COOKIE_NAME` constant and `SIDEBAR_COOKIE_MAX_AGE` — replace these with a `SIDEBAR_STORAGE_KEY` constant and use localStorage instead.

Key changes:
- Replace `document.cookie = ...` with `localStorage.setItem(SIDEBAR_STORAGE_KEY, ...)`
- Replace cookie parsing for initial state with `localStorage.getItem(SIDEBAR_STORAGE_KEY)`
- Remove `SIDEBAR_COOKIE_MAX_AGE`

- [ ] **Step 2: Rewrite `Layout.tsx`**

Replace the entire file. The new layout uses shadcn Sidebar + sticky navbar pattern:

```tsx
import { Suspense } from "react";
import { Outlet, useNavigate, NavLink } from "react-router-dom";
import { useTranslation } from "react-i18next";
import {
  Check,
  CircleUserRound,
  Languages,
  LogOut,
  Palette,
} from "lucide-react";
import {
  Sidebar,
  SidebarContent,
  SidebarFooter,
  SidebarGroup,
  SidebarGroupContent,
  SidebarHeader,
  SidebarInset,
  SidebarMenu,
  SidebarMenuButton,
  SidebarMenuItem,
  SidebarProvider,
  SidebarTrigger,
} from "@/components/ui/sidebar";
import { Separator } from "@/components/ui/separator";
import { Button } from "@/components/ui/button";
import {
  DropdownMenu,
  DropdownMenuContent,
  DropdownMenuItem,
  DropdownMenuSeparator,
  DropdownMenuSub,
  DropdownMenuSubContent,
  DropdownMenuSubTrigger,
  DropdownMenuTrigger,
} from "@/components/ui/dropdown-menu";
import ThemedSuspense from "@/features/ThemedSuspense";
import { useAuthReducer } from "@/atoms/auth";
import { useTheme } from "@/components/theme-provider";
import { THEMES } from "@/atoms/theme";
import { routes, type RouteConfig } from "@/routes";
import { cn } from "@/lib/utils";

/* --- Implementation notes for the agent ---
 *
 * 1. Filter routes with `nav: true` from routes.ts for sidebar items.
 * 2. Use NavLink with SidebarMenuButton for active-state styling.
 * 3. The sticky navbar header goes inside SidebarInset, above Outlet.
 * 4. Port the account dropdown (theme/language/logout) from the old NavBar.tsx
 *    into the sticky header's right side.
 * 5. Port the brand area ("Aurora" + dot) into SidebarHeader.
 * 6. Preserve the auth check + websocket init from the current Layout.tsx
 *    (useEffect with token, initializeWebSocket, getToken).
 */
```

The structure should match the spec's code skeleton:
```tsx
<SidebarProvider>
  <Sidebar>
    <SidebarHeader>{/* Brand: dot + "Aurora" */}</SidebarHeader>
    <SidebarContent>
      <SidebarGroup>
        <SidebarGroupContent>
          <SidebarMenu>
            {sidebarRoutes.map((route) => (
              <SidebarMenuItem key={route.key}>
                <SidebarMenuButton asChild isActive={/* use NavLink */}>
                  <NavLink to={route.fullPath}>
                    <route.icon />
                    <span>{t(route.labelKey)}</span>
                  </NavLink>
                </SidebarMenuButton>
              </SidebarMenuItem>
            ))}
          </SidebarMenu>
        </SidebarGroupContent>
      </SidebarGroup>
    </SidebarContent>
    <SidebarFooter>{/* optional: theme/lang if desired here */}</SidebarFooter>
  </Sidebar>
  <SidebarInset>
    <header className="sticky top-0 z-30 flex h-16 shrink-0 items-center gap-4 border-b border-border backdrop-blur-md bg-background/80 px-4">
      <SidebarTrigger className="-ml-1" />
      <Separator orientation="vertical" className="mr-2 h-4" />
      <div className="flex-1" />
      {/* Account dropdown with theme picker, language, logout */}
    </header>
    <main className="flex-1 overflow-auto">
      <Suspense fallback={<ThemedSuspense />}>
        <div className="flex w-full flex-col items-center">
          <div className="container p-4 md:p-6">
            <Outlet />
          </div>
        </div>
      </Suspense>
    </main>
  </SidebarInset>
</SidebarProvider>
```

Preserve the auth/websocket `useEffect` from the current `Layout.tsx` (lines 22-41).

- [ ] **Step 3: Delete old layout files**

```bash
rm frontend/src/features/layout/SideBar.tsx
rm frontend/src/features/layout/NavBar.tsx
```

- [ ] **Step 4: Verify build compiles**

```bash
cd frontend && npx tsc --noEmit 2>&1 | head -50
```

Expected: May show errors in page components that still reference deleted modal/notification imports — these will be fixed in wave tasks. The layout shell itself should compile.

- [ ] **Step 5: Commit**

```bash
git add -A frontend/src/
git commit -m "feat: rebuild layout shell with shadcn Sidebar + sticky navbar"
```

---

## Task 7: Wave 1 — Auth pages

**Files:**
- Modify: `frontend/src/features/auth/Login.tsx`
- Modify: `frontend/src/features/auth/CreateAccount.tsx`
- Modify: `frontend/src/features/auth/EmailPasswordForm.tsx`
- Modify: `frontend/src/features/theme/ThemeSwitch.tsx`
- Modify: `frontend/src/features/theme/ThemeMenuItems.tsx`
- Modify: `frontend/src/features/i18n/LanguageSwitch.tsx`
- Modify: `frontend/src/features/i18n/LanguageMenuItems.tsx`

- [ ] **Step 1: Update `EmailPasswordForm.tsx`**

This is the shared form component. Update:
- Component imports to use new radix-nova `Input`, `Button`, `Label` (import paths stay the same `@/components/ui/*`, but verify variant/size prop compatibility)
- The `addNotification` calls were already migrated to `toast()` in Task 4
- Check for any `font-heading` classes (already replaced globally in Task 3)
- Remove any remaining `useModal` imports if present

- [ ] **Step 2: Update `Login.tsx`**

Update component imports and variant props. This page is a simple form wrapper — minimal changes expected beyond ensuring new button/input sizing looks correct.

- [ ] **Step 3: Update `CreateAccount.tsx`**

Same treatment as Login.tsx.

- [ ] **Step 4: Update `ThemeSwitch.tsx` and `LanguageSwitch.tsx`**

These components are used in the Layout sidebar footer and auth pages. Update variant/size props to match radix-nova components (`Button`, `DropdownMenu`). `ThemeSwitch.tsx` can be simplified to a light/dark toggle since we're on zinc only for now, though the full theme infrastructure still works.

Also review `ThemeMenuItems.tsx` and `LanguageMenuItems.tsx` for the same prop compatibility.

- [ ] **Step 5: Smoke test the auth flow**

```bash
cd frontend && npx tsc --noEmit 2>&1 | grep -E "Login|CreateAccount|EmailPasswordForm|ThemeSwitch|LanguageSwitch"
```

Expected: No type errors in these files.

- [ ] **Step 6: Commit**

```bash
git add frontend/src/features/auth/ frontend/src/features/theme/ frontend/src/features/i18n/
git commit -m "feat(wave-1): migrate auth, theme, and language pages to radix-nova components"
```

---

## Task 8: Wave 2 — Server pages

**Files:**
- Modify: `frontend/src/features/server/ServerList.tsx`
- Modify: `frontend/src/features/server/ServerRow.tsx`
- Modify: `frontend/src/features/server/ServerCard.tsx`
- Modify: `frontend/src/features/server/ServerContainer.tsx`
- Modify: `frontend/src/features/server/ServerStat.tsx`
- Modify: `frontend/src/features/server/ServerPortsStat.tsx`
- Modify: `frontend/src/features/server/ServerTrafficStat.tsx`
- Modify: `frontend/src/features/server/ServerSSHStat.tsx`
- Modify: `frontend/src/features/server/ServerInfoModal.tsx`
- Modify: `frontend/src/features/server/chart/Chart.tsx`
- Modify: `frontend/src/features/server/chart/Sparkline.tsx`
- Modify: `frontend/src/hooks/useServerItem.ts`

- [ ] **Step 1: Convert `ServerInfoModal.tsx` to inline Dialog**

This is the most complex change in this wave. The file currently uses `useModal()` (line 7) and `ModalShell` (line 11). Convert to:
- Remove `useModal` import, add `useState`
- Remove `ModalShell` import, add `Dialog`, `DialogContent`, `DialogHeader`, `DialogTitle`, `DialogFooter` imports
- Replace `ModalShell` wrapper with `DialogContent` + `DialogHeader` + `DialogFooter`
- The modal is currently opened via `useModal().open("serverInfo", {...})` from `ServerList.tsx` — change to pass `open`/`onOpenChange` props or move the Dialog inline into the parent
- The `notify()` calls were already migrated to `toast()` in Task 4

- [ ] **Step 2: Update `ServerList.tsx`**

- Remove `useModal` import (line 30)
- Add `useState` for controlling the ServerInfoModal dialog
- Render `<ServerInfoModal>` inline with Dialog wrapper
- Update `Card`, `Badge`, `Button` variant props if needed

- [ ] **Step 3: Update `useServerItem.ts`**

- Remove `useModal` import (line 2)
- This hook uses `useModal` to open server info — refactor to return a callback or state setter instead

- [ ] **Step 4: Update `ServerRow.tsx` and `ServerCard.tsx`**

- Update `Card`, `Badge`, `Button`, `DropdownMenu` variant/size props
- `ServerRow.tsx` notification migration was done in Task 4
- Remove any remaining `useModal` references

- [ ] **Step 5: Update stat components**

Update `ServerStat.tsx`, `ServerPortsStat.tsx`, `ServerTrafficStat.tsx`, `ServerSSHStat.tsx`:
- Update `Card` imports and variant props
- These are mainly presentational — minimal changes

- [ ] **Step 6: Update `ServerContainer.tsx`**

Update any layout-related imports. This is the parent route component.

- [ ] **Step 7: Review chart components**

`chart/Chart.tsx` and `chart/Sparkline.tsx` use Recharts. Check if they should wrap in the new `chart.tsx` component from the zip. If they work as-is, leave them.

- [ ] **Step 8: Type-check server pages**

```bash
cd frontend && npx tsc --noEmit 2>&1 | grep -E "features/server/"
```

Expected: No type errors.

- [ ] **Step 9: Commit**

```bash
git add frontend/src/features/server/ frontend/src/hooks/useServerItem.ts
git commit -m "feat(wave-2): migrate server pages to radix-nova + inline dialogs"
```

---

## Task 9: Wave 3 — Port & User pages

**Files:**
- Modify: `frontend/src/features/port/ServerPorts.tsx`
- Modify: `frontend/src/features/port/PortCard.tsx`
- Modify: `frontend/src/features/port/PortSelectCard.tsx`
- Modify: `frontend/src/features/port/PortUsersCard.tsx`
- Modify: `frontend/src/features/port/PortFunctionModal.tsx`
- Modify: `frontend/src/features/port/PortRestrictionModal.tsx`
- Modify: `frontend/src/features/port/restriction/PortExpiration.tsx`
- Modify: `frontend/src/features/user/Users.tsx`
- Modify: `frontend/src/features/user/ServerUsers.tsx`

- [ ] **Step 1: Convert `PortFunctionModal.tsx` to inline Dialog**

Remove `useModal` + `ModalShell` pattern. Replace with `Dialog` + `DialogContent` + local state. The parent (`PortCard.tsx` or `ServerPorts.tsx`) should render the Dialog inline.

- [ ] **Step 2: Convert `PortRestrictionModal.tsx` to inline Dialog**

Same pattern as PortFunctionModal.

- [ ] **Step 3: Update `ServerPorts.tsx`**

- Remove `useModal` import (line 13)
- Add inline Dialog rendering for port modals
- Update `Card`, `Select`, `Badge` props

- [ ] **Step 4: Update `PortCard.tsx` and `PortSelectCard.tsx`**

- Remove `useModal` imports
- Update component variant props

- [ ] **Step 5: Update `PortUsersCard.tsx` and `PortExpiration.tsx`**

Update imports and props.

- [ ] **Step 6: Update `Users.tsx` and `ServerUsers.tsx`**

Update `Table`, `Button`, `Badge` imports and props.

- [ ] **Step 7: Type-check**

```bash
cd frontend && npx tsc --noEmit 2>&1 | grep -E "features/(port|user)/"
```

- [ ] **Step 8: Commit**

```bash
git add frontend/src/features/port/ frontend/src/features/user/
git commit -m "feat(wave-3): migrate port and user pages to radix-nova + inline dialogs"
```

---

## Task 10: Wave 4 — Files & Deployments

**Files:**
- Modify: `frontend/src/features/file/FileCenter.tsx`
- Modify: `frontend/src/features/file/FileCenterContainer.tsx`
- Modify: `frontend/src/features/file/FileCard.tsx`
- Modify: `frontend/src/features/file/FileRow.tsx`
- Modify: `frontend/src/features/file/FileModal.tsx`
- Modify: `frontend/src/features/file/FilePreviewModal.tsx`
- Modify: `frontend/src/features/deployment/DeploymentList.tsx`
- Modify: `frontend/src/features/deployment/DeploymentStatusBadge.tsx`
- Modify: `frontend/src/features/deployment/DeployModal.tsx`
- Modify: `frontend/src/features/deployment/DeploymentDetailModal.tsx`
- Modify: `frontend/src/features/deployment/BindingModal.tsx`

- [ ] **Step 1: Convert file modals to inline Dialog**

`FileModal.tsx` and `FilePreviewModal.tsx` — remove `ModalShell`, replace with `Dialog`/`DialogContent`. `FilePreviewModal` may suit `Sheet` for a slide-over preview.

- [ ] **Step 2: Update `FileCenter.tsx`, `FileCard.tsx`, `FileRow.tsx`**

- Remove `useModal` imports
- Render file dialogs inline
- Update `Card`, `Table`, `Badge` props

- [ ] **Step 3: Convert deployment modals to inline Dialog**

`DeployModal.tsx`, `DeploymentDetailModal.tsx`, `BindingModal.tsx` — same pattern: remove `ModalShell` + `useModal`, replace with inline `Dialog`.

- [ ] **Step 4: Update `DeploymentList.tsx` and `DeploymentStatusBadge.tsx`**

- Remove `useModal` imports
- Render deployment dialogs inline
- Update `Badge`, `Table` props

- [ ] **Step 5: Update `FileCenterContainer.tsx`**

Minor layout updates.

- [ ] **Step 6: Type-check**

```bash
cd frontend && npx tsc --noEmit 2>&1 | grep -E "features/(file|deployment)/"
```

- [ ] **Step 7: Commit**

```bash
git add frontend/src/features/file/ frontend/src/features/deployment/
git commit -m "feat(wave-4): migrate file and deployment pages to radix-nova + inline dialogs"
```

---

## Task 11: Wave 5 — Services & Misc pages

**Files:**
- Modify: `frontend/src/features/service-editor/ServiceListPage.tsx`
- Modify: `frontend/src/features/service-editor/ServiceEditorPage.tsx`
- Modify: `frontend/src/features/service-editor/AuthoringJsonPanel.tsx`
- Modify: `frontend/src/features/service-editor/FormPreviewPanel.tsx`
- Modify: `frontend/src/features/service-editor/CompileOutputPanel.tsx`
- Modify: `frontend/src/features/service-editor/ParamEditorPanel.tsx`
- Modify: `frontend/src/features/service-editor/ParamList.tsx`
- Modify: `frontend/src/features/service-editor/ParamTypeEditor.tsx`
- Modify: `frontend/src/features/service-editor/EmitConfigEditor.tsx`
- Modify: `frontend/src/features/service-editor/ValidationEditor.tsx`
- Modify: `frontend/src/features/service-editor/ConditionEditor.tsx`
- Modify: `frontend/src/features/service-editor/UIConfigEditor.tsx`
- Modify: `frontend/src/features/service-editor/fields/TextField.tsx`
- Modify: `frontend/src/features/service-editor/fields/SelectField.tsx`
- Modify: `frontend/src/features/service-editor/fields/CheckboxField.tsx`
- Modify: `frontend/src/features/service-editor/fields/TextAreaField.tsx`
- Modify: `frontend/src/features/service-editor/fields/ListField.tsx`
- Modify: `frontend/src/features/service-editor/fields/ObjectField.tsx`
- Modify: `frontend/src/features/service-editor/fields/FieldsRenderer.tsx`
- Modify: `frontend/src/features/service-editor/fields/FieldShell.tsx`
- Modify: `frontend/src/features/service-editor/fields/FieldError.tsx`
- Modify: `frontend/src/features/about/About.tsx`
- Modify: `frontend/src/features/layout/Themes.tsx`
- Modify: `frontend/src/features/layout/Error.tsx`
- Modify: `frontend/src/features/layout/NoMatch.tsx`
- Modify: `frontend/src/features/layout/Hero.tsx`

- [ ] **Step 1: Update `ServiceListPage.tsx`**

- Remove `useModal` import (line 17)
- Update `Card`, `Button`, `Table` variant props

- [ ] **Step 2: Update `ServiceEditorPage.tsx` and panels**

Update `Tabs`, `Card`, `Button`, `Input` props across `ServiceEditorPage.tsx`, `AuthoringJsonPanel.tsx`, `FormPreviewPanel.tsx`, `CompileOutputPanel.tsx`, `ParamEditorPanel.tsx`. Consider using `Resizable` from the new components for the editor split-pane if applicable.

- [ ] **Step 3: Update service editor field components**

Update all files in `fields/` — these use `Input`, `Select`, `Checkbox`, `Textarea`, `Label`. Import paths stay the same; verify variant/size props are compatible.

- [ ] **Step 4: Update remaining service editor files**

`ParamList.tsx`, `ParamTypeEditor.tsx`, `EmitConfigEditor.tsx`, `ValidationEditor.tsx`, `ConditionEditor.tsx`, `UIConfigEditor.tsx` — update `Input`, `Select`, `Button` imports and props.

- [ ] **Step 5: Update `Themes.tsx`**

Simplify since we're on zinc light/dark only. The theme picker page can still show the available themes from the `THEMES` array, but only light/dark will be visually distinct.

- [ ] **Step 6: Update misc pages**

`About.tsx`, `Error.tsx`, `NoMatch.tsx`, `Hero.tsx` — minimal updates.

- [ ] **Step 7: Type-check**

```bash
cd frontend && npx tsc --noEmit 2>&1 | grep -E "features/(service-editor|about|layout)/"
```

- [ ] **Step 8: Commit**

```bash
git add frontend/src/features/service-editor/ frontend/src/features/about/ frontend/src/features/layout/
git commit -m "feat(wave-5): migrate service editor, about, and misc pages to radix-nova"
```

---

## Task 12: Update shared utilities

**Files:**
- Modify: `frontend/src/features/ui/PageHeader.tsx`
- Modify: `frontend/src/features/ui/PageSection.tsx`
- Modify: `frontend/src/features/ui/EmptyState.tsx`
- Modify: `frontend/src/features/DataLoading.tsx`
- Modify: `frontend/src/features/Paginator.tsx`
- Modify: `frontend/src/features/ThemedSuspense.tsx`

- [ ] **Step 1: Update `PageHeader.tsx`**

Update to use new component primitives. The `font-heading` class was already replaced globally in Task 3.

- [ ] **Step 2: Update `PageSection.tsx` and `EmptyState.tsx`**

Update component imports and props.

- [ ] **Step 3: Update `DataLoading.tsx`**

May use new `Spinner` or `Skeleton` from radix-nova set.

- [ ] **Step 4: Update `Paginator.tsx`**

Update `Button` variant props.

- [ ] **Step 5: Update `ThemedSuspense.tsx`**

Simplify loading state with new `Spinner` or `Skeleton`.

- [ ] **Step 6: Commit**

```bash
git add frontend/src/features/ui/ frontend/src/features/DataLoading.tsx frontend/src/features/Paginator.tsx frontend/src/features/ThemedSuspense.tsx
git commit -m "feat: update shared UI utilities to radix-nova primitives"
```

---

## Task 13: Final cleanup and dependency removal

**Files:**
- Modify: `frontend/package.json`
- Verify: entire `frontend/src/`

- [ ] **Step 1: Remove unused font packages**

```bash
cd frontend && npm uninstall @fontsource-variable/noto-sans @fontsource-variable/space-grotesk
```

- [ ] **Step 2: Full type-check**

```bash
cd frontend && npx tsc --noEmit
```

Expected: No errors.

- [ ] **Step 3: Verify no dead imports**

```bash
grep -r "atoms/modal" frontend/src/
grep -r "atoms/notification" frontend/src/
grep -r "atoms/layout" frontend/src/
grep -r "ModalShell" frontend/src/
grep -r "useModal" frontend/src/
grep -r "ModalManager" frontend/src/
grep -r "drawerOpenAtom" frontend/src/
grep -r "font-heading" frontend/src/
grep -r "next-themes" frontend/src/
grep -r "notificationManager" frontend/src/
```

Expected: No matches for any of these.

- [ ] **Step 4: Verify the `modal/` directory is gone**

```bash
ls frontend/src/features/modal/ 2>&1
```

Expected: "No such file or directory"

- [ ] **Step 5: Build check**

```bash
cd frontend && npm run build
```

Expected: Build succeeds.

- [ ] **Step 6: Commit**

```bash
git add -A frontend/
git commit -m "chore: final cleanup — remove unused fonts and verify no dead imports"
```
