---
name: new-domain
description: Create a new backend domain (bounded context) following the platform's reference structure — all layers, protocols, dishka providers, registration, migration, tests. Use when the user asks to add a new backend domain / module / entity with CRUD ("new domain", "add entity X", "new domain/module").
---

# New backend domain

Reference: [rules/BACKEND_STRUCTURE.md](../../../rules/BACKEND_STRUCTURE.md) (must be
read before you start). Laws: [rules/BACKEND_RULES.md](../../../rules/BACKEND_RULES.md) §3–§10.

## Step 0 — align with the user
- **Domain name** — plural, snake_case (`orders`); **entity** — singular (`Order`).
- The set of model fields, relationships with other domains, which operations are needed (full CRUD or a subset).
- If the domain depends on another domain — only through its public protocol or an event
  (rules 41–44), NEVER through importing another domain's internals.

## Step 1 — create the structure

```
apps/<domain>/
├── router.py  ├── schemas.py  ├── use_case.py  ├── services.py
├── repositories.py  ├── models.py  ├── exceptions.py  └── providers.py
```

A simple domain uses files; **the second entity in a domain → the layer expands into a folder with
one file per entity; a use case is one file per user story; a file is ≤ 400 lines (CI gate);
`utils.py` dumping grounds inside a domain are forbidden** (rules 46–48). `events.py` — only
if the domain publishes events (then → the `new-event` skill).

## Step 2 — write bottom-up
1. **`models.py`** — inherit `Base` (+`TimestampMixin`); table name plural, PK `id`
   (UUID v7), booleans `is_*`, `server_default`, `index=True` on FKs, `lazy="raise"`.
2. **Import the model into `migrations/env.py`** — otherwise autogenerate won't see it.
3. **`schemas.py`** — Read/Create/Update (repository boundary) + Request/Response
   (use case boundary); `camelCase` outward, `extra="forbid"`.
4. **`repositories.py`** — `Protocol` + `Impl(GenericRepository)`; DTO in / DTO out;
   does NOT commit (flush only).
5. **`services.py`** — `Protocol` + `Impl`; injects protocols.
6. **`use_case.py`** — one callable class per user story; returns a Response schema;
   permission/ownership checks go here (BOLA, rule 82).
7. **`providers.py`** — all bindings `provide(Impl, provides=Protocol, scope=Scope.REQUEST)`.
8. **`router.py`** — `@inject` + `FromDishka[UseCaseProtocol]`, no business logic.

## Step 3 — registration (both are mandatory)
1. Router → into the router aggregator.
2. `<Domain>Provider` → into `make_async_container(...)` in `bootstrap.py`.

## Step 4 — migration and tests
- `uv run alembic revision --autogenerate -m "..."` → review the generated DDL by eye.
- Tests mirror `src/`: unit tests on the use case/service (in-memory repository through the protocol),
  an integration test on the repository.

## Verification before finishing
- [ ] import-linter / typecheck (pyright) / tests pass
- [ ] The domain does not import other domains' internals (rules 42–44)
- [ ] The ORM does not leave the repository; the repository does not commit (rules 25, 27)
- [ ] Router and Provider are registered; `/docs` shows the endpoints
- [ ] Cross-check the diff against the "Layers", "DI", "Domains", and "DB Models" sections of
      [rules/RULES_CHECKLIST.md](../../../rules/RULES_CHECKLIST.md)
