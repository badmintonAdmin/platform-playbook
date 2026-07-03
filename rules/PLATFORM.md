# Platform Principles (cross-cutting)

> A top-level ruleset shared by **all** platform services and clients. It defines the
> boundaries, contracts, and conventions that backend, frontend, and any other client
> (mobile app, third-party integrations, CLI) are required to follow.
>
> Backend technology details live in [BACKEND_RULES.md](BACKEND_RULES.md),
> [BACKEND_RULES_PRODUCTION.md](BACKEND_RULES_PRODUCTION.md),
> [BACKEND_STRUCTURE.md](BACKEND_STRUCTURE.md). Client-side details are in
> [FRONTEND_STRUCTURE.md](FRONTEND_STRUCTURE.md). This document sits above them and **takes
> precedence** in a conflict: if a specific rule contradicts a platform principle, the
> principle wins (or the specific rule is corrected).

---

## 1. API-first and multi-client

**The platform's core principle: the API is a product, not an implementation detail of a single frontend.**

The same HTTP API serves **several independent clients**: the web frontend, the mobile
app, third-party integrations, internal services. From this it follows that:

1. **The API treats no client as "its own."** The API makes no assumptions like "this is
   called by our Next.js." There are no endpoints "for a specific page," no fields "because
   it's convenient for this particular screen." The contract describes the **domain**, not
   the UI.
2. **The backend knows nothing about rendering.** No HTML, no server-side markup for a
   client, no "let's put a ready-made button label here." The client decides for itself how
   to display data.
3. **The client never touches the database and knows nothing about the backend layers.** The
   only contract between client and server is the **published API** (see §3). The client
   communicates solely through it, with no direct access to storage.
4. **The contract is versioned, not silently broken.** A mobile app user may go months
   without updating — the older client version must keep working. An incompatible change =
   **a new path version** (`/api/v2/...`), see §4.
5. **Formats are machine-readable and stable.** Dates are ISO-8601 in UTC; money is integer
   minor units (kopecks/cents) or a string with a currency; identifiers are stable and
   opaque. No locale/timezone/formatting on the API side — that is the client's
   responsibility.

**Check for every new endpoint:** "Could a mobile app and a third-party integration that
don't exist yet call this without pain?" If the answer depends on knowing about a specific
frontend, the endpoint is designed incorrectly.

---

## 2. A single layered model (across the whole platform)

The boundaries are identical at every level: **dependencies point inward, toward the abstraction.**

```
┌──────────────────────────────────────────────────────────────┐
│  Clients (web / mobile / integrations)                        │
│      know ONLY the published API contract (§3)                │
├──────────────────────────────────────────────────────────────┤
│  Backend: Router → UseCase → Service → Repository → I/O       │
│      depend on Protocol, not on implementation (see BACKEND_RULES) │
└──────────────────────────────────────────────────────────────┘
```

- **The "client ↔ server" boundary** = the API contract (§3). Only DTOs cross it.
- **The boundaries inside the backend** = the clean-architecture layers (see
  [BACKEND_RULES.md §4](BACKEND_RULES.md)).
- **Portability rule:** changing an implementation on one side of a boundary must not change
  the code on the other. Swapping the database does not touch the client; redesigning a
  screen does not touch the API; changing the framework does not touch the business logic.

---

## 3. The API contract — the only seam between client and server

The contract comes first and is the **source of truth** for all clients.

### The schema as the contract
- The backend publishes an **OpenAPI schema** (FastAPI generates it from Pydantic DTOs and
  routes automatically). The schema is a release artifact, not a side effect.
- **Client types are generated from OpenAPI**, not written by hand. A type mismatch between
  client and server is a build-time bug, not a runtime one.
- No client keeps "its own view" of the server's fields alongside the contract.
- **A process, not discipline:** generated types are committed; CI checks their freshness
  (regeneration + `git diff --exit-code`) and **catches breaking schema changes in the PR**
  (an OpenAPI diff against main), not in production.

### Casing at the boundary
- **Outward (the JSON API) — `camelCase`.** Inside Python — `snake_case`. The conversion is
  done by the Pydantic schemas (`alias_generator` + `populate_by_name`), not by manual
  mapping.
- The client works with `camelCase` and **does not rename fields** for its own needs.

### A unified error format
All API errors use **one machine-readable envelope**, identical for all clients:

```jsonc
{
  "error": "ModelNotFound",         // stable error-class code (for branching on the client)
  "message": "User not found",       // human-readable, NOT for parsing
  "details": { "field": "email" }    // optional: context (which fields, which limits)
}
```

- The HTTP status carries the semantics (404/403/409/422/429/5xx), the body carries the
  details.
- **The client branches on `error` (the code), not on `message`** — the message may change
  and be localized.
- The mapping of domain errors → HTTP is centralized on the backend (see
  [BACKEND_RULES.md §11](BACKEND_RULES.md)). The envelope format is uniform across the whole
  platform.

### What the API returns and what it does not
- **Does not return** internal entities/ORM objects, technical fields, secrets, or PII
  beyond what is needed.
- **Returns** domain DTOs. Secrets are **write-only** (accepted on write, never returned;
  masked in responses, logs, and the UI).

---

## 4. Versioning and compatibility

- The path is versioned with a prefix: **`/api/v1/...`**.
- **Backward-compatible** changes (adding an optional field, a new endpoint) go into the
  current version.
- **Incompatible** ones (removing/renaming a field, changing a type, tightening validation,
  changing semantics) require **a new path version**. The old version lives until an agreed
  decommissioning (for mobile clients — with a margin).
- The compatibility rule is the same SemVer principle as for code (see
  [BACKEND_RULES.md §14](BACKEND_RULES.md)): breaking → MAJOR → a new API version.
- **A tolerant client:** clients ignore unknown fields in a response (they don't break when
  the server adds a field). This is a mandatory precondition for the safe evolution of the
  contract.

---

## 5. Identity and sessions — across all clients

The authentication model must work **identically** for web and mobile (see also
[BACKEND_RULES_PRODUCTION.md §1](BACKEND_RULES_PRODUCTION.md)).

- **A token model (JWT) as the basis for multi-client.** A short-lived access token +
  a refresh token with rotation and the ability to revoke. Mobile clients don't handle
  cookies the way a browser does.
- **Web** may use an httpOnly cookie on top of the same token model; **mobile /
  integrations** use `Authorization: Bearer`. The server accepts both delivery methods for
  the same token — but **this is one identity mechanism**, not two parallel ones.
- **Cookie delivery brings CSRF protection with it:** `SameSite=Lax/Strict` on the cookie +
  a CSRF token on mutating requests (double-submit or a header). CSRF does not concern
  Bearer clients — the token is not sent automatically by the browser.
- **Authorization (whether an action is allowed) is always on the backend, in the domain
  layer.** The client may hide a button, but the server makes the decision. No client is a
  trusted boundary.
- **Resource ownership checks (object-level) belong in every UseCase.** Authentication ≠
  the right to a specific object (OWASP API #1, see PRODUCTION §1–2).

---

## 6. Configuration and environments

- **12-factor:** configuration comes from the environment, not from the code and not from
  the repository. The same artifact (image/bundle) is run across the `local` → `staging` →
  `production` environments; only the config changes.
- **The repository contains only `.env.example`** (a template without values). Real values
  come from the environment; production secrets come from a secrets manager, not from a file
  on disk (PRODUCTION §14).
- **No client hardcodes the API base URL** — only from the environment configuration.
- Feature flags and behavior toggles go through config/feature-flags, not through code
  branches "per environment."

---

## 7. Observability across boundaries

- A **Correlation / Request ID** travels **through the entire chain**: the client generates
  or the server assigns an `X-Request-ID`, it ends up in every log statement and in the
  error response and is propagated into outgoing calls (see PRODUCTION §3). A single id lets
  you reconstruct a request's path from the client all the way to the database.
- **Structured logs** (JSON), without PII/secrets, to stdout.
- **Product analytics** (user events) is separated from technical observability
  (logs/metrics/traces) and is not mixed with it.

---

## 8. Naming and vocabulary

- **A single ubiquitous language** across the whole platform: one entity is named the same
  in the database, in DTOs, in API paths, and in the client. `User` is `user` everywhere,
  not `account` in one place and `member` in another.
- The code conventions for each layer are in the corresponding files (backend naming in
  [BACKEND_STRUCTURE.md](BACKEND_STRUCTURE.md); API paths and casing — here in §3).
- Domain boundaries (**bounded contexts**) on the backend = separate domains in `apps/`
  (see [BACKEND_RULES.md §3](BACKEND_RULES.md)); their public parts in the API do not leak
  domain implementation details.

---

## 9. Scale calibration

The platform is designed to be **large in structure, not in load**. The design point is
**hundreds of users, not millions**. From this it follows that:

- **We optimize for changeability and simplicity, not throughput.** Clean architecture,
  domain isolation, and contracts — at full scale (they are about manageability); horizontal
  scaling "like the big players" — no.
- **A modular monolith is the default.** Domain isolation (see BACKEND_RULES §4) preserves
  the option to split out a service later, but we **do not split out services "for future
  growth."**
- **A service is split out only for a strong reason:** a different release cycle, fault
  isolation, a fundamentally different technology/load. "It's trendier" is not a reason.
- **Don't drag in heavy infrastructure without need:** sharding, multi-region, clusters, a
  service mesh — overkill. PostgreSQL + Redis + one broker cover the needs at this scale.
- The contract rules (§3–4, §10) are nonetheless followed **in full** — they are about
  evolution and compatibility, not about load.

---

## 10. Event contracts (broker / queues)

If services communicate through a broker (Kafka / RabbitMQ / NATS / Redis), **an event is
just as much a public contract as the HTTP API**, with the same strictness about schema and
compatibility.

### Schema and naming
- The event payload is a **versioned Pydantic schema** that lives in the producer domain
  (`apps/<domain>/events.py`). The consumer depends **only on the event schema**, not on the
  producer's internals.
- The event name: `<domain>.<entity>.<action>` in the **past tense** — an event describes an
  accomplished fact, not a command: `orders.order.created`, `users.user.deactivated`.
- Schema evolution is **additive only** (new optional fields). A breaking change = **a new
  event version** (`orders.order.created.v2`); the old one lives as long as there are
  consumers.

### The event envelope
Every message carries a standard envelope:

```jsonc
{
  "messageId": "0195…",          // UUID — for deduplication by the consumer
  "eventName": "orders.order.created",
  "eventVersion": 1,
  "occurredAt": "2026-07-02T12:00:00Z",   // UTC, the moment the fact occurred
  "correlationId": "…",           // cross-cutting chain id (see §7)
  "payload": { … }                // versioned schema
}
```

### Reliability (see PRODUCTION §9, §11)
- Publishing is done **only through a Transactional Outbox** (in the same transaction as the
  domain change); dual-write is forbidden.
- Delivery is at-least-once ⇒ **every consumer is idempotent** (dedup by `messageId`);
  "poison" messages go to a DLQ with retry/backoff.

---

## Summary of platform "do / don't"

**DO**
- Design the API from the domain, not from the screen; keep it usable by a mobile and a
  third-party client from day one.
- The only client↔server contract is the published OpenAPI; client types are generated from
  it.
- `camelCase` at the boundary, a unified error envelope, client branching on the error code.
- Version the API by path; clients are tolerant of unknown fields.
- One token-based identity model for all clients; authorization on the server.
- Config from the environment; a correlation id through the entire chain.

**DON'T**
- Build endpoints "for a page"; return ready-made markup/localized strings.
- Let the client know about the backend layers/database, bypassing the API.
- Break the contract without a new version; branch the client on the error message text.
- Keep two parallel authentication models for web and mobile.
- Hardcode the API base URL; return secrets/ORM entities to the outside.
