---
name: new-frontend-section
description: Add a section/page/surface to the web client according to the frontend rules — route in a route group, components organized by folders, data via apiFetch and types from OpenAPI, a11y. Use when the user asks to add a frontend page / section / screen / surface ("new page", "dashboard section", "screen", "new page/section").
---

# New frontend section

Rules: [rules/FRONTEND_STRUCTURE.md](../../../rules/FRONTEND_STRUCTURE.md);
platform contract: [rules/PLATFORM.md](../../../rules/PLATFORM.md) §3–§5.

## Step 0 — determine the location
- **Which surface?** User-facing (`/app`), admin (`/app/admin`), public
  (`(marketing)` / `(auth)`). Don't mix purposes (rule 111): admin content belongs
  only in the admin surface.
- **Which route group?** Public/protected — explicitly; protected is covered by `middleware.ts`.
- Role-based visibility — a frontend gate is UX only; the real check is done by the backend
  (rule 13).

## Step 1 — data
- Which endpoints are needed? If they don't exist — first define the contract on the backend
  (→ skill `new-endpoint`); the frontend doesn't invent the shape of the data.
- Types — **from the generated OpenAPI types** (rule 6); don't write DTOs by hand.
- All calls — through the single `apiFetch` (rule 104): no bare `fetch`, no
  hardcoded URLs (rule 14).
- Server state — in the project's chosen mechanism (RSC-fetch / query library);
  don't copy API responses into the global store (rule 106).

## Step 2 — components
- The page is **thin**: composition of components (rule 108).
- Server component by default; `"use client"` — only on interactive leaves
  (rule 105).
- Reusable primitives → `components/ui/` (no domain logic);
  section-specific parts → the folder of its surface (rule 110).

## Step 3 — states and errors
- Required states: loading, empty, error, success. Errors — from the single envelope:
  show `message`, branch on the `error` code (rule 9).
- Forms: client-side validation for UX + mapping of server-side field errors from `details`
  (rule 113).
- No data / metrics not instrumented → show "Pending", don't make things up
  (rule 114).
- Secrets in the UI — write-only: mask them, an empty field preserves the previous value
  (rule 85).

## Step 4 — a11y (gate, rule 115)
- Semantic HTML, labels on fields, keyboard navigation, visible focus, contrast.

## Verification
- [ ] No bare `fetch` / hardcoded URLs / manual DTOs
- [ ] Client boundary is minimal; the page is thin
- [ ] All 4 UI states; errors by envelope code
- [ ] Cross-check against the "Frontend" section of
      [rules/RULES_CHECKLIST.md](../../../rules/RULES_CHECKLIST.md)
