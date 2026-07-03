---
name: new-endpoint
description: Add an endpoint to an existing backend domain — a vertical slice from user story to route through all layers (use case → service → repository → provider → router). Use when the user asks to add an endpoint / operation / API method to an existing domain ("add an endpoint", "need an endpoint", "new endpoint/route").
---

# New endpoint (vertical slice)

Rules: [rules/BACKEND_RULES.md](../../../rules/BACKEND_RULES.md) §4–§7, §11;
[rules/PLATFORM.md](../../../rules/PLATFORM.md) §1, §3–§5.

## Step 0 — formulate the user story
One endpoint = one user story = one UseCase (rule 21). Formulate: "who does what,
and what do they get". Apply the multi-client test (rules 1–2): the endpoint
describes a problem domain, not the screen of a specific frontend.

## Step 1 — design the contract
- Path: `/{entities}` (plural), method by semantics; path under the current API version.
- Request/Response schemas: `camelCase` externally, `extra="forbid"`, no
  `id`/`role`/`is_admin` from the client (rule 32).
- Changing an existing contract? Check compatibility: breaking → new path
  version (rule 10).
- Errors: which domain exceptions are raised, which HTTP codes they map to (the base
  ones already exist in `core` — don't create duplicates).

## Step 2 — implement bottom-up
1. **Repository** — need a new query? Add a method to the `Protocol` + `Impl`
   (DTO in/out, flush without commit). Required parameters before `*`, flags after.
2. **Service** — reusable logic/invariants. If the logic is one-off, you can put it
   directly in the use case; don't bloat the service artificially.
3. **UseCase** — `Protocol` + `Impl`, `async def __call__`; permission and resource
   ownership checks go HERE (rules 81–82); returns the Response schema.
4. **Provider** — bind the new use case in the domain's `providers.py`.
5. **Router** — `@inject`, `FromDishka[UseCaseProtocol]`, no logic; paginate
   large lists with keyset/cursor (rule 97).

## Step 3 — transactions and side effects
- Mutating a single aggregate — the request transaction (the provider commits itself).
- Multiple repositories — Unit of Work (rule 29).
- Publishing an event — via the outbox in the same transaction (→ skill `new-event`).
- Critical POST (payment, order creation) — consider `Idempotency-Key` (rule 95).

## Step 4 — tests
- Unit on the use case (in-memory protocols): happy path + permission/validation errors.
- Test for mapping the domain error to an HTTP code and the `{error, message, details}` envelope.

## Verification
- [ ] No logic in the route; permission/ownership check present in the use case
- [ ] Contract is compatible or the version was bumped; `/docs` is correct
- [ ] Transaction boundary chosen deliberately (request / UoW)
- [ ] Review the diff against the "Layers", "Errors", and "Security" sections of
      [rules/RULES_CHECKLIST.md](../../../rules/RULES_CHECKLIST.md)
