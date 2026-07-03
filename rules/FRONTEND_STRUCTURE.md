# Frontend Structure (Reference)

> Canonical structure and baseline rules for the platform's web client (Next.js App Router).
> The frontend is **one of the clients** of the shared API (on equal footing with the mobile
> app and integrations), so it obeys the platform contract from
> [PLATFORM.md](PLATFORM.md): it communicates only through the published API, generates its
> types from OpenAPI, uses `camelCase`, and relies on the unified error envelope.
>
> Stack: Next.js App Router + TypeScript. The styling system and component library are
> determined by the product; the rules below do not depend on them. The package manager is pnpm.
>
> This is a baseline. Deeper rules (state, data fetching, forms, performance, a11y) will be
> added separately.

---

## 1. The Frontend Is a Client, Not a "Companion to the Backend"

- The web frontend is **not privileged**: the API is unaware of it and does not adapt to its
  screens (see [PLATFORM.md §1](PLATFORM.md)). Everything the frontend needs, it obtains through
  the same contract as the mobile app.
- **The only contract is the published API.** No direct access to the database, to internal
  services, or to backend layers that bypass the API.
- **DTO types are generated from the backend's OpenAPI schema**, not written by hand. A
  hand-written type that duplicates the contract is a source of drift.
- Authorization logic lives on the server. The frontend may hide an element based on role, but
  **does not treat this as security** (see [PLATFORM.md §5](PLATFORM.md)).

---

## 2. Surfaces: Several Surfaces, One Shell

The application may have **several surfaces** with different audiences and purposes, sharing a
common dashboard shell. Keep them conceptually and structurally separate.

Typical pattern:

| Surface | Route | Audience | Purpose |
|---|---|---|---|
| **User** | `/app` | ordinary user | managing their own data |
| **Admin** | `/app/admin` | operator/account owner | managing the application within the account |

Principles:
- **One surface = one responsibility.** Do not mix user functions with administrative ones in a
  single screen.
- **Surface visibility is gated by role**, while the backend **must** verify access on its own
  side (a frontend gate is UX only, not security).
- If a **cross-account operator console** becomes necessary, it is a **separate, third** surface,
  not an admixture to the existing ones.

---

## 3. Route Structure (App Router route groups)

Route groups `( … )` organize code without changing the URL. Public and protected are explicit.

```
app/
├── (marketing)/        # PUBLIC — landing pages (many; often edited, incl. by AI agent; /, /l/[slug])
├── (auth)/             # PUBLIC — minimal auth layout (login / signup / …)
├── (app)/              # PROTECTED (middleware) — dashboard shell
│   ├── layout.tsx      #   guards + shell chrome
│   ├── app/            #   user surface (/app)
│   └── app/admin/      #   admin surface (/app/admin)
├── api/                # route handlers (BFF: only where a server-side intermediary is needed)
├── layout.tsx          # root layout
└── globals.css         # global styles (if the styling model is global)
```

- A single dashboard route, without duplicated/atypical segments (do not proliferate `/dashboard`
  and `/app` at the same time).
- `middleware.ts` protects the protected group → sends an unauthenticated user to
  `/login?return_to=…`.
- `app/api/` (route handlers) are **only a BFF intermediary** (proxying a secret, aggregating for
  SSR). Do not move business logic there: it lives in the backend.
- **Public landing pages** (`(marketing)`) are isolated from `(app)`: they do not import
  dashboard code, sessions, or permissions; they can be moved to a separate deployment later. A
  landing page is thin presentation + a call to a **stable** public lead-intake endpoint
  (form → `apiFetch` → `POST /leads`). The intake logic is on the backend; editing a landing page
  (including by an AI agent) does not touch the contract. No visual builder/CMS is introduced.

---

## 4. Rendering Boundaries (Server / Client Components)

- **Server Component by default.** `"use client"` is used only where interactivity/browser APIs
  are needed (state, effects, handlers).
- Keep the client boundary **as low as possible** in the tree (leaf interactive components), so
  as not to pull unnecessary things into the client.
- **Secrets and private keys are server-only.** Nothing beyond public configuration
  (`NEXT_PUBLIC_*`) should end up in the client bundle.
- Load data as close as possible to where it is used; do not pass server objects into the client
  unless necessary.

---

## 5. Component Organization

```
components/
├── ui/          # shared presentational primitives (Button, Card, Modal, …) —
│                #   thin wrappers over the styling system, without business logic
├── <surface>/   # components for the sections of a specific surface (dashboard, admin, …)
└── landing/     # sections of marketing pages
```

- Shared → `ui/`. Surface-specific → into that surface's folder.
- **Pages are thin**: they compose components and contain no business logic or scattered direct
  fetches.
- `ui/` primitives are unaware of the domain (reusable across surfaces).

### Granularity (anti-god-files)

- **One component = one file; file ≤ 300 lines — a CI gate** (generated code — allowlist). A page
  is a thin composition, targeting ≤ 150 lines.
- When a component accumulates state/logic, the logic is extracted into a **hook in a separate
  file** (`use-<name>.ts`) rather than bloating the JSX.
- **A catch-all `utils.ts` is forbidden**: `lib/` holds modules organized by meaning
  (`lib/format-money.ts`, `lib/date.ts`), not a dumping ground.

---

## 6. `lib/` — Client Infrastructure

- **`lib/env.ts`** — the single source of the API base URL (server `API_URL` / browser
  `NEXT_PUBLIC_API_URL`). **Never** hardcode the API base in a page/component
  (see [PLATFORM.md §6](PLATFORM.md)).
- **`lib/api-client.ts`** — the **only** `apiFetch`: **all** API calls go through it. It
  centralizes:
  - identity delivery (cookie/Bearer — see [PLATFORM.md §5](PLATFORM.md));
  - parsing of the **unified error envelope** (`error`/`message`/`details`) → a typed client
    error;
  - branching on the `error` **code**, not on the `message` text;
  - the reaction to `401` (→ `/login`) and forwarding of `X-Request-ID` (see PLATFORM §7).
- **`lib/types.ts`** — DTOs **generated** from the backend's OpenAPI (not hand-written duplicates).
- **`lib/session.ts`** — helpers for the current user/session.
- **`lib/validation.ts`** — form validation (zod or an equivalent) with clear messages; client-side
  validation **duplicates** server-side validation, it does not replace it.

---

## 7. Data, Errors, Forms (baseline)

- **Data fetching** goes only through `apiFetch`. No "bare" `fetch` calls with manual URL/casing
  scattered across components.
- **Server data is not duplicated in a global store.** **One** server-state mechanism is chosen
  per project (RSC fetch and/or a query library with cache and revalidation) — and all API data
  lives in it. A global client store is only for pure UI state (open panels, form drafts), not for
  copies of API responses.
- **Error handling** relies on the unified envelope: we show the user a human-readable message and
  branch logic by code. Unknown response fields are ignored (tolerant client, PLATFORM §4).
- **Forms**: client-side validation for UX + trust in server-side validation as the source of
  truth; field errors are mapped from the envelope's `details`.
- **Secrets in the UI are write-only**: they are masked, an empty field preserves the previous
  value, and raw keys are neither shown nor logged (see PLATFORM §3).
- **Uninstrumented data** (metrics/figures that do not exist yet) is shown as "Pending" rather than
  inventing values.

---

## 8. Accessibility (baseline)

- Semantic HTML, keyboard-accessible interactive elements, correct `aria-*`, visible focus,
  sufficient contrast.
- Forms: associated `label`s, clear error messages, loading/error state announced to assistive
  technologies.
- Accessibility is a **release gate**, not decoration (the product may tighten this section).

---

## Frontend "do / don't" Summary

**DO**
- Reach the API only through the single `apiFetch`; types from OpenAPI; `camelCase` at the boundary.
- Keep surfaces separate; gate visibility by role (protection is on the backend).
- Server Component by default; the client boundary lower in the tree.
- Unified error envelope, branching by code; secrets write-only; a11y as a gate.
- API base from the environment configuration.

**DON'T**
- Access the database/internal layers bypassing the API; treat a frontend gate as security.
- Duplicate DTOs by hand; branch on the error message text.
- Hardcode the API base URL; drag secrets/server objects into the client bundle.
- Put business logic into pages or into the `app/api/` BFF.
