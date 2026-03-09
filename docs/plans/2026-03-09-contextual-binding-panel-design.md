# Contextual Binding Panel — Design

**Date:** 2026-03-09
**Status:** Approved
**Goal:** Replace the generic BindingModal with a contextual property panel that adapts based on whether it was opened from a file card or a service definition.

---

## Core Concept

The BindingModal becomes a small, focused modal that shows "the other side" — from a file you see services, from a service you see files.

## From a File Card (fileId provided)

- **Header:** File name + version as title (e.g. "gost-v3.0.2")
- **Body:** Simple list of bound services. Each row: service title, service key badge, version. Clicking a row selects it (visual highlight).
- **Footer area:**
  - Inline dropdown of available services + "Add" button
  - "Remove" button appears when a row is selected (with confirmation)

## From a Service Definition (serviceId provided)

- **Header:** Service title as modal title (e.g. "Gost Relay")
- **Body:** Simple list of bound files. Each row: file name, version, file size.  Clicking a row selects it (visual highlight).
- **Footer area:**
  - Inline dropdown of available executable files + "Add" button
  - "Remove" button appears when a row is selected (with confirmation)

## Shared Behavior

- Small modal (`max-w-md`)
- Empty state: "No bindings yet" with add dropdown still visible
- The "known" side (file or service) is in the header, never in the list
- No table — clean list with subtle dividers
- Selected row gets `bg-primary/10` highlight + "Remove" button appears
- Confirmation before removing a binding
