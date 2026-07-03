# Platform Rules Checklist

> A compressed digest of the entire rulebook — one rule = one line. For code review, PR checklists,
> and quick cross-checks. Details and rationale are in the full documents:
> [PLATFORM.md](PLATFORM.md), [BACKEND_RULES.md](BACKEND_RULES.md),
> [BACKEND_STRUCTURE.md](BACKEND_STRUCTURE.md),
> [BACKEND_RULES_PRODUCTION.md](BACKEND_RULES_PRODUCTION.md),
> [FRONTEND_STRUCTURE.md](FRONTEND_STRUCTURE.md).
> On conflict, the full document higher in the hierarchy wins (see [README.md](README.md)).

## Platform / API

1. The API is a product for many clients (web, mobile, integrations), not an implementation detail of a single frontend.
2. Endpoints are designed from the domain, never for a specific screen.
3. The platform is designed for hundreds of users, not millions: simplicity and changeability matter more than horizontal scaling; a modular monolith by default.
4. A separate service is split out only for a strong reason (a different release cycle, failure isolation), not "to grow into"; heavy infrastructure (sharding, mesh, multi-region) is overkill.
5. The only client↔server contract is the published OpenAPI; nobody bypasses the API.
6. Client types are generated from OpenAPI, never written by hand; the generated code is committed, and CI checks it is up to date.
7. A contract breaking change is caught by an OpenAPI diff on the PR, not in production.
8. Outward-facing: `camelCase`, internal Python: `snake_case`; Pydantic schemas convert between them.
9. An error is a single envelope `{error, message, details}`; the client branches on the `error` code, not on the text.
10. The API is versioned by path (`/api/v1/`); a breaking change = a new version, the old one lives until it is retired.
11. The client is tolerant: it ignores unknown fields in the response and does not crash.
12. One token-based identity model for all clients (JWT: short-lived access + refresh with rotation).
13. Authorization is always on the server, in the domain layer; the frontend gate is UX only.
14. Config from the environment (12-factor); the API base URL is never hardcoded.
15. A correlation/request ID flows through the whole chain: client → logs → errors → outgoing calls and events.
16. One entity is named the same everywhere: DB, DTO, API, events, client (ubiquitous language).

## Layers and dependencies (backend)

17. Layers strictly downward: `Router → UseCase → Service → Repository`; injecting "upward" is forbidden.
18. Depend always on an abstraction (`Protocol`), never on an implementation (`Impl`).
19. Interfaces via `typing.Protocol` (not ABC); naming: `XProtocol` / `XImpl`.
20. No business logic in the controller; only the UseCase is injected into the controller.
21. One UseCase = one user story = one endpoint; it is a callable class (`async def __call__`).
22. A UseCase returns a response schema, not a domain/ORM object.
23. A Service is reusable domain logic; it injects other services and repositories.
24. A repository injects only the session.
25. Only DTOs (Pydantic) travel between layers; a DTO goes into the repository and a DTO comes out — the ORM model never rises above the repository.
26. The repository encapsulates any I/O: DB, HTTP, Redis, broker.
27. The repository never commits — only flushes; the transaction is owned by the boundary (the request or the UoW).
28. Default is one transaction per request: commit in the session provider on success, rollback on exception.
29. A mutation across several repositories goes only through a Unit of Work; the UoW owns the transaction.
30. Complex reads (reports, dashboards) go through a separate read-only query service: SQL → straight to a Response DTO, bypassing the domain layers; mutations may not do this.
31. Schemas are split by purpose: Request/Response (use case boundary), Read/Create/Update (repository boundary).
32. Pydantic schemas are strict: `extra="forbid"`; `id`/`role`/`is_admin` are not accepted from the client.

## DI (dishka)

33. DI is dishka only; `fastapi.Depends` as an IoC mechanism and hand-rolled factories are forbidden.
34. The `Impl → Protocol` binding is done via `provide(..., provides=Protocol)` in the domain's `providers.py`.
35. `Scope.APP` — singletons (engine, sessionmaker, pools, config); `Scope.REQUEST` — session, repositories, services, use cases.
36. Session and transaction via a generator provider (`yield` + commit/rollback).
37. Swapping an implementation (Postgres → in-memory) = changing one binding, the business logic is untouched.
38. The container is assembled in `bootstrap.py` and closed in the lifespan.
39. A new domain is registered with two actions: the router into the aggregator + a Provider into the container.

## Domains and structure

40. Every domain in `apps/` has the same reference structure (router, schemas, use_case, services, repositories, models, exceptions, events, providers).
41. A domain's public surface is only its `router`, service/use case protocols (via DI), and `events.py`; everything else is private.
42. A domain does not import another domain's internals (models, repositories, `Impl`); interaction is a public protocol via DI or a domain event.
43. `core` does not import from `apps`; `apps` imports `core` — never the other way around.
44. Domain isolation is checked automatically — `import-linter` in CI, not code review by eye.
45. What is common to all domains lives in `core/` (base models, generic CRUD, exceptions, providers).
46. A layer as a single file — only while there is one entity; a second entity → the layer becomes a folder with a file per entity; a use case is a file per user story.
47. A file ≤ 400 lines is a CI gate (exceptions: migrations, generated code); if exceeded, split it, do not raise the limit.
48. Junk drawers are forbidden: `utils.py`/`helpers.py` are not created in a domain; what is shared goes into `core` as a module with a descriptive name.
49. Do not put scripts at the root — only a Typer CLI.
50. Tests mirror the structure of `src/`.

## DB models

51. A model is singular (`User`), a table is plural (`users`).
52. The PK is always `id`; preference: UUID v7 → int → UUID v4.
53. Boolean fields are prefixed with `is_`.
54. `created_at`/`updated_at` — always; all time is UTC; always `server_default`.
55. Native ENUMs in the DB are forbidden — the enum lives on the Python side, in the DB it is a string/int.
56. An entity uses typed columns, not a JSON junk drawer; `JSONB` only as an auxiliary field (`extra`) for irregular data; queried fields are separate indexable columns, and a matured field is promoted from `extra` via a migration.
57. `index=True` on every FK; `passive_deletes=True` on the relationship.
58. Lazy loading is forbidden in async: `lazy="raise"`, relations are loaded explicitly (`selectinload`).
59. Invariants expressible in the DB go in the DB: CHECK / UNIQUE / NOT NULL, not only Pydantic.
60. Soft delete: `deleted_at` + a default filter in the repository + a partial unique index.
61. Concurrent updates — optimistic locking (`version_id`); critical spots — `SELECT FOR UPDATE`.

## Migrations

62. Model First: models → autogenerate.
63. All models are imported in `env.py`, otherwise autogenerate does not see them.
64. A migration name carries the date: `year_month_day_slug`.
65. Breaking DDL — only expand → contract across separate releases: each release's schema is compatible with the previous release's code.

## Errors

66. Base exceptions live in `core`, by HTTP meaning (`ModelNotFound → 404`); domain ones inherit from them.
67. Before the controller — only business errors, after it — only HTTP; the controller is the conversion boundary.
68. Mapping to HTTP — centralized exception handlers, one per base class.
69. Domain exceptions — the `Error` suffix.

## Events and broker

70. An event is just as much a public contract as the API: the same strictness about schema and compatibility.
71. Event name: `domain.entity.action` in the past tense (`orders.order.created`) — an accomplished fact, not a command.
72. The payload is a versioned Pydantic schema in `apps/<domain>/events.py`; evolution is additive only, a breaking change → a new event version.
73. Event envelope: `messageId`, `eventName`+`eventVersion`, `occurredAt` (UTC), `correlationId`.
74. The consumer depends only on the event schema, not on the producer's internals.
75. Publishing — only through the Transactional Outbox (in the same transaction as the domain change); dual-write is forbidden.
76. The consumer is idempotent: dedup by `messageId` (inbox); "poison" messages — to the DLQ with retry/backoff.

## Background work

77. Workers, consumers, and the scheduler use the same DI container; a REQUEST scope is opened per message/job.
78. A cron task runs under a distributed lock — N replicas do not run it N times.
79. A job is idempotent and reduces to invoking a use case; there is no business logic in the scheduler/consumer code.

## Security

80. Passwords — Argon2id only.
81. Permission checks — in the UseCase/Service, not in the router; `CurrentUser` — via a REQUEST-scope provider.
82. Ownership of a specific resource is checked in every UseCase (BOLA).
83. Rate limiting on auth endpoints; security headers; CORS — an explicit allowlist, never `*` + credentials.
84. Web cookie session — `SameSite` + a CSRF token on mutating requests; Bearer clients are not affected by CSRF.
85. Secrets are write-only: not returned, masked, not logged.
86. Production secrets — from a secrets manager, `.env` — local only.
87. In CI/pre-commit: gitleaks, bandit, pip-audit.
88. Soft delete ≠ erasure: PII on a deletion request or on retention expiry — hard delete or anonymization; retention periods are defined per entity.

## Operations

89. Logs — structured JSON to stdout; PII and secrets are not logged; `print` is forbidden.
90. Timeouts on everything: HTTP client, DB, external calls — "no timeout" is forbidden.
91. Graceful shutdown on SIGTERM: let in-flight work finish, close pools and the container.
92. Liveness and readiness — separate probes (`/health/live` without dependencies, `/health/ready` — with them).
93. Dockerfile — multi-stage, non-root, base image by digest.
94. Retry — only on idempotent calls, with backoff and jitter; cascades — a circuit breaker.
95. Critical POSTs — an `Idempotency-Key`: a repeat returns the stored response.
96. Cache — cache-aside, always with a TTL, invalidation by events, stampede protection with a lock.
97. Pagination of large lists — keyset/cursor, not offset.

## Tests and quality

98. The pyramid: many unit tests (domain via Protocol + in-memory), fewer integration, minimal e2e.
99. Integration tests — on testcontainers; data factories — polyfactory; rollback — savepoint.
100. Pyright strict + mypy + Ruff + pre-commit — mandatory.
101. Ruff complexity rules are enabled (C901, PLR0912/0913/0915); a long function is refactored, not added to the exceptions.
102. Conventional Commits + semantic-release; the version is computed from the commits.

## Frontend

103. The frontend is one of the API's clients, not a privileged one.
104. All API calls go through the single `apiFetch`; bare `fetch` calls scattered across components are forbidden.
105. Server Component by default; `"use client"` — only for interactivity, with the boundary pushed lower down the tree.
106. Server data is not duplicated in the global store: one mechanism for server state (RSC and/or a query library); the global store is for pure UI state only.
107. Secrets — server only; into the client bundle — only `NEXT_PUBLIC_*`.
108. Pages are thin: composition of components, without business logic.
109. One component = one file (≤ 300 lines — a CI gate); logic that outgrows it goes into a hook in its own file; a catch-all `utils.ts` is forbidden.
110. Shared primitives live in `ui/`, unaware of the domain; specifics go in per-surface folders.
111. Surfaces (user/admin) are kept separate; a cross-account console is a third surface, not an admixture.
112. `app/api/` (BFF) — a mediator only, business logic lives in the backend.
113. Client-side validation duplicates server-side validation, it does not replace it.
114. Uninstrumented data is shown as "Pending", not fabricated.
115. Accessibility is a release gate: semantics, keyboard, focus, contrast.
