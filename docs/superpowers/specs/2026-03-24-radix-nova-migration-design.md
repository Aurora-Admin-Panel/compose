# Radix-Nova UI Migration Design

## Summary

Replace all 20 existing shadcn/ui components with the full radix-nova style component set (58 components after excluding duplicates and unused toast files), adopt radix-nova as the new design language, rebuild the layout shell on the shadcn Sidebar primitive, replace the Jotai modal manager with inline Dialog/Sheet usage, switch notifications to Sonner, and migrate all pages in waves.

## Decisions

| Decision | Choice | Rationale |
|---|---|---|
| Style direction | Radix-nova replaces Aurora personality | Clean break, accept new design language entirely |
| Theming | Zinc light/dark active; `.theme-*` scaffolding kept empty; ThemeProvider infrastructure preserved | Future theme addition without current maintenance burden |
| `"use client"` directives | Strip all | Vite doesn't need them; cleaner code |
| Toast/notification system | Sonner only | Modern, minimal, shadcn's direction |
| Sidebar | Adopt shadcn sidebar primitive | Replace custom SideBar.tsx + NavBar.tsx |
| Modal pattern | Inline Dialog/Sheet with local state | Replace Jotai ModalManager pattern |
| Migration strategy | Drop-and-replace, then page waves | Single-owner project; fastest path with least lingering mess |
| Migration order | Foundation → Layout → Auth → Server → Port/User → Files/Deploy → Services/Misc | Waves ordered by complexity; auth first as smoke test |

## Section 1: Foundation — Component Drop & CSS Reset

### Components

Copy all `.tsx` files from the radix-nova zip `components/ui/` into `frontend/src/components/ui/`, replacing the existing 20, with these exclusions:

- **Exclude `toast.tsx` and `toaster.tsx`** — We are using Sonner only; these are the older Radix Toast system and would create a conflicting dual-toast setup.
- **Exclude `use-mobile.tsx` and `use-toast.ts`** from `components/ui/` — These are hooks that are duplicated in the zip's `hooks/` directory. Copy the `hooks/` versions instead to avoid import confusion.

Copy from the zip's `hooks/` directory into `src/hooks/`:
- `use-mobile.ts`

Do NOT copy `use-toast.ts` — Sonner is the active toast system and this file would be dead code.

This results in **58 component files** in `src/components/ui/`.

Modifications to every component file:
- Strip `"use client"` directives
- Verify `@/lib/utils` and `@/components/ui/*` import paths match the existing Vite alias setup (they already do)

### Sonner adaptation (CRITICAL)

The zip's `sonner.tsx` imports `useTheme` from `next-themes`. This project uses a custom `ThemeProvider` at `@/components/theme-provider`. Rewrite the import:

```diff
- import { useTheme } from "next-themes"
+ import { useTheme } from "@/components/theme-provider"
```

The custom `useTheme` returns `{ theme, resolvedTheme, setTheme }` where `resolvedTheme` is a `ThemeName` like `"aurora-classic"` or `"midnight"` — not `"light"` or `"dark"`. Derive Sonner's theme from the `THEMES` config:

```typescript
import { useTheme } from "@/components/theme-provider"
import { THEMES } from "@/atoms/theme"

const { resolvedTheme } = useTheme()
const themeConfig = THEMES.find((t) => t.name === resolvedTheme)
const sonnerTheme = themeConfig?.colorScheme === "dark" ? "dark" : "light"
```

This looks up the theme's `colorScheme` property (`"light"` or `"dark"`) which correctly handles all theme names.

### CSS (`index.css`)

- Adopt the zip's `app/globals.css` theme variable structure (the `@theme inline` block with `radius-sm` through `radius-4xl`)
- Replace color variables with the zip's zinc light/dark palette as the active `:root` and `.dark` defaults
- **Add `--destructive-foreground`** variable to both `:root` and `.dark` blocks — the zip defines this and several new components reference it (toast, alert-dialog, button destructive variant). Use the zip's values.
- Keep the `.theme-*` class scaffolding as empty shells (no color definitions) for future theme additions. The `ThemeProvider` infrastructure (class toggling, `'d'` keyboard shortcut, cross-tab sync, localStorage persistence) remains functional — it just applies to the single zinc light/dark pair for now.
- **`font-heading` removal strategy:** Remove the `@fontsource-variable/space-grotesk` and `@fontsource-variable/noto-sans` imports and the `--font-heading` / `--font-sans` CSS variable overrides. Then do a global find-and-replace of `font-heading` → `font-sans` across all files that use it as a Tailwind class. This affects ~15 files including `dialog.tsx`, `card.tsx`, `alert.tsx`, `sheet.tsx`, `PageHeader.tsx`, `EmptyState.tsx`, `ServerCard.tsx`, `FileCard.tsx`, and service editor panels.
- **Remove `@plugin './plugins/tailwind-mix.js'`** and `@tailwindcss/typography` — no files use the `prose` class.
- Keep `@import "shadcn/tailwind.css"`

### Notifications

**Sonner integration:**
- Add `<Toaster />` (Sonner) to the root provider stack in `main.tsx`
- Remove `features/Notification.tsx`

**Notification migration (CRITICAL):**
The current codebase uses notifications in two ways:

1. **Imperative `notify()` calls** (outside React) — `graphql.ts` (Apollo error link) uses the non-hook `notify()` from `atoms/notification.ts`
2. **Hook-based `useNotificationsReducer` / `addNotification` calls** (inside React) — used by `features/server/ServerRow.tsx` and `features/auth/EmailPasswordForm.tsx`

Sonner's `toast()` function supports both patterns — it can be called imperatively from anywhere, or from within components.

Migration steps:
- Replace all `notify({...})` calls in `graphql.ts` with `toast.error(message)` / `toast.success(message)`
- Replace all `addNotification({...})` calls in `ServerRow.tsx` and `EmailPasswordForm.tsx` with `toast()` calls
- Update `App.tsx` to remove the `<Notification />` render and its lazy import (also remove `<ModalManager />` render)
- Delete `atoms/notification.ts`
- Delete `atoms/notificationManager.ts` (older EventTarget-based system, currently unused)

### `lib/utils.ts`

Already has the same `cn()` function — no change needed.

## Section 2: Layout Shell — Sidebar & Navigation

### Remove

- `features/layout/SideBar.tsx` (rebuilt on shadcn Sidebar primitive)
- `features/layout/NavBar.tsx` (rebuilt as a sticky header inside `SidebarInset`)
- `features/ui/ModalShell.tsx`
- `features/modal/ModalManager.tsx`
- `features/modal/ConfirmationModal.tsx`
- `features/Notification.tsx`

### Sidebar persistence adaptation (CRITICAL)

The zip's `sidebar.tsx` persists collapse state via `document.cookie` — this is a Next.js SSR pattern. In this Vite SPA, replace the cookie mechanism with `localStorage` (or reuse the existing `drawerOpenAtom` from `atoms/layout.ts` if appropriate). The sidebar should read initial state from localStorage on mount and write state changes back.

### Rebuild `Layout.tsx` — Sidebar + Sticky Navbar

The current layout pattern is **sidebar (left) + sticky top navbar (top of content area)**. This pattern is preserved using the shadcn Sidebar primitive:

```
┌──────────┬──────────────────────────────┐
│          │  Sticky NavBar               │
│ Sidebar  │  [SidebarTrigger] [spacer] [account dropdown] │
│          ├──────────────────────────────┤
│  nav     │                              │
│  items   │  <Outlet /> (page content)   │
│          │                              │
│          │                              │
│ footer:  │                              │
│ theme/   │                              │
│ lang     │                              │
└──────────┴──────────────────────────────┘
```

**Sidebar (left):**
- `SidebarProvider` + `Sidebar` + `SidebarContent` + `SidebarGroup` + `SidebarMenu` as structural skeleton
- Port existing nav items from `routes.ts` into `SidebarMenuItem` entries
- Theme switch and language switch move into `SidebarFooter`
- Mobile: shadcn sidebar handles responsive collapse natively (sheet-based overlay)

**Sticky NavBar (top of content area):**
- Lives inside `SidebarInset`, above the `<Outlet />`
- Sticky header with `sticky top-0 z-30 backdrop-blur` styling (same pattern as current `NavBar.tsx`)
- Left side: `SidebarTrigger` (hamburger to toggle sidebar on mobile, collapse on desktop)
- Left side (optional): `Separator` + `Breadcrumb` for page context
- Right side: account `DropdownMenu` with theme picker submenu, language submenu, and logout — same structure as current `NavBar.tsx`
- The hamburger no longer needs `drawerOpenAtom` — `SidebarTrigger` handles toggle internally

**Structure in code:**
```tsx
<SidebarProvider>
  <Sidebar>
    <SidebarContent>{/* nav items */}</SidebarContent>
    <SidebarFooter>{/* theme + lang switches */}</SidebarFooter>
  </Sidebar>
  <SidebarInset>
    <header className="sticky top-0 z-30 flex h-16 items-center gap-4 border-b backdrop-blur bg-background/80 px-4">
      <SidebarTrigger />
      <div className="flex-1" />
      {/* account dropdown */}
    </header>
    <main>
      <Outlet />
    </main>
  </SidebarInset>
</SidebarProvider>
```

### Jotai cleanup

- Remove modal-related atoms
- Remove `drawerOpenAtom` from `atoms/layout.ts` (sidebar trigger handles toggle internally now)
- Auth atoms, theme atoms, and other app state atoms remain untouched

### Provider stack in `main.tsx`

- Add `<Toaster />` (Sonner)
- Remove any modal-manager provider
- Keep `ApolloProvider` → `ThemeProvider` → `TooltipProvider` → `HelmetProvider` → `Suspense` order

## Section 3: Page Migration Waves

### Wave 1 — Auth pages (smoke test)

- `Login.tsx` — Update form to use new `Input`, `Button`, `Label`
- `CreateAccount.tsx` — Same treatment
- `EmailPasswordForm.tsx` — Shared form component, update imports

### Wave 2 — Server pages (most complex)

- `ServerList.tsx` / `ServerRow.tsx` / `ServerCard.tsx` — Update `Card`, `Badge`, `Button`, `DropdownMenu` variants
- `ServerContainer.tsx` — Parent layout for server detail area
- `ServerStat.tsx` / `ServerPortsStat.tsx` / `ServerTrafficStat.tsx` / `ServerSSHStat.tsx` — Stat cards
- `ServerInfoModal.tsx` — Convert from ModalManager to inline `Dialog`
- `chart/Chart.tsx` / `chart/Sparkline.tsx` — Mostly untouched unless wrapping in new `chart.tsx`

### Wave 3 — Port & User pages

- `ServerPorts.tsx` / `PortCard.tsx` / `PortSelectCard.tsx` / `PortUsersCard.tsx` — Update `Card`, `Select`, `Badge`
- `PortFunctionModal.tsx` / `PortRestrictionModal.tsx` — Convert to inline `Dialog`
- `port/restriction/PortExpiration.tsx` — Form field updates
- `Users.tsx` / `ServerUsers.tsx` — Update `Table`, `Button`, `Badge`

### Wave 4 — Files & Deployments

- `FileCenter.tsx` / `FileCenterContainer.tsx` / `FileCard.tsx` / `FileRow.tsx` — Update `Card`, `Table`, `Badge`
- `FileModal.tsx` / `FilePreviewModal.tsx` — Convert to inline `Dialog`/`Sheet`
- `DeploymentList.tsx` / `DeploymentStatusBadge.tsx` — Update `Badge`, `Table`
- `DeployModal.tsx` / `DeploymentDetailModal.tsx` / `BindingModal.tsx` — Convert to inline `Dialog`

### Wave 5 — Services & Misc

- `ServiceListPage.tsx` — Update `Card`, `Button`, `Table`
- `ServiceEditorPage.tsx` + panels (`AuthoringJsonPanel`, `FormPreviewPanel`, `CompileOutputPanel`, `ParamEditorPanel`) — Update `Tabs`, `Card`, `Button`, `Input`. `Resizable` from zip could replace custom split-pane.
- Service editor field components — full list: `fields/TextField.tsx`, `fields/SelectField.tsx`, `fields/CheckboxField.tsx`, `fields/TextAreaField.tsx`, `fields/ListField.tsx`, `fields/ObjectField.tsx`, `fields/FieldsRenderer.tsx`, `fields/FieldShell.tsx`, `fields/FieldError.tsx`, `fields/index.ts`. Update `Input`, `Select`, `Checkbox`, `Textarea`, `Label`.
- Additional service editor files: `ParamList.tsx`, `ParamTypeEditor.tsx`, `EmitConfigEditor.tsx`, `ValidationEditor.tsx`, `ConditionEditor.tsx`, `UIConfigEditor.tsx`, `useDynamicForm.tsx`, `serviceAdapter.ts`, `builderUtils.ts`, `formUtils.ts`, `constants.ts`. These use `Input`, `Select`, `Button` and need import updates.
- `About.tsx` / `Themes.tsx` — Simple pages, minimal updates. Themes page may need simplification since we're dropping to zinc light/dark only.
- `Error.tsx` / `NoMatch.tsx` / `Hero.tsx` — Layout utility pages

### Shared utilities (updated across all waves)

- `features/ui/PageSection.tsx`, `PageHeader.tsx`, `EmptyState.tsx` — Update to new primitives
- `features/DataLoading.tsx` — Update `Skeleton` or `Spinner` usage
- `features/Paginator.tsx` — Update `Button` variants
- `features/ThemedSuspense.tsx` — Update loading state

### Modal conversion pattern (applied in every wave)

1. Remove atom-based open/close logic
2. Add local `useState` or use `Dialog` with `DialogTrigger`
3. Replace `<ModalShell>` with `<DialogContent>` + `<DialogHeader>` + `<DialogFooter>`
4. For destructive confirmations, use `AlertDialog` instead of `Dialog`

## Section 4: Cleanup & Dependencies

### Package.json additions

- `sonner` (toast)
- `vaul` (drawer dependency)
- `embla-carousel-react` (carousel dependency)
- `react-day-picker` + `date-fns` (calendar dependency)
- `input-otp`
- `react-resizable-panels`

### Package.json removals (after all waves)

- `@fontsource-variable/noto-sans`
- `@fontsource-variable/space-grotesk`

### Files to delete after all waves

- `features/modal/ModalManager.tsx`
- `features/modal/ConfirmationModal.tsx`
- `features/ui/ModalShell.tsx`
- `features/Notification.tsx`
- `features/layout/SideBar.tsx`
- `features/layout/NavBar.tsx`
- `atoms/notification.ts`
- `atoms/notificationManager.ts`
- `drawerOpenAtom` from `atoms/layout.ts`
- Modal-related Jotai atoms
- `plugins/tailwind-mix.js`

### Files to update

- `App.tsx` — Remove `<Notification />` and `<ModalManager />` renders and their lazy imports
- `graphql.ts` — Replace `notify()` calls with Sonner `toast()` calls
- `features/server/ServerRow.tsx` — Replace `addNotification()` with `toast()`
- `features/auth/EmailPasswordForm.tsx` — Replace `addNotification()` with `toast()`

### Files to keep but review

- `features/theme/ThemeSwitch.tsx` — Adapt for sidebar footer; simplify to light/dark toggle only
- `features/i18n/LanguageSwitch.tsx` — Adapt for sidebar footer
- `features/ThemedSuspense.tsx` — May simplify with `Spinner` or `Skeleton`
- `components/theme-provider.tsx` — Keep as-is; infrastructure works for zinc light/dark and future themes

### Untouched

- All GraphQL queries (`src/queries/`)
- All Jotai atoms except modal/notification-related (`src/atoms/`)
- All existing hooks (`src/hooks/`) except new additions
- `routes.ts`
- `graphql.ts` — Only change is replacing `notify()` calls with Sonner `toast()` calls
- `tailwind-safelist.ts` — Still needed for service editor dynamic grid layouts
- Backend — no changes

## Appendix: Known Transition Period

Between Section 2 (layout rebuild) and Wave 1 completion, the entire app will be visually broken — the old layout shell is removed but pages haven't been updated yet. This is acceptable for a single-owner project. Each wave restores functionality to its page group.
