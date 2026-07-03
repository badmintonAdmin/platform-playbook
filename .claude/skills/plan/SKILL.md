---
name: plan
description: Turn a product or feature idea into a phased, rules-compliant delivery plan written to plans/, then hand off to the build skills so development runs almost turnkey. Use when the user brings an idea and wants a plan before coding ("plan this", "let's build X", "break this into phases", "spec out the project", "план", "распиши по фазам").
---

# Plan a product (idea → phased delivery)

Turn a raw idea into a plan a full-stack build can execute nearly turnkey, honoring the
**whole ruleset**. The plan lives in `plans/`; each phase is then executed with the build
skills ([new-domain](../new-domain/SKILL.md), [new-endpoint](../new-endpoint/SKILL.md),
[new-event](../new-event/SKILL.md), [new-frontend-section](../new-frontend-section/SKILL.md))
and gated by [rules-review](../rules-review/SKILL.md).

**Read first:** [rules/README.md](../../../rules/README.md) (the map),
[rules/PLATFORM.md](../../../rules/PLATFORM.md),
[rules/RULES_CHECKLIST.md](../../../rules/RULES_CHECKLIST.md),
[rules/GIT_AND_DELIVERY.md](../../../rules/GIT_AND_DELIVERY.md). This is a
**FastAPI + Next.js** stack — plan within it, not around it.

## Step 0 — understand the idea (align with the user)
- **What** is the product/feature; **who** are the users and clients (web, mobile,
  integrations — the API serves all of them, [PLATFORM §1](../../../rules/PLATFORM.md)).
- **Core user stories** — the vertical slices. Separate must-have (MVP) from later.
- **Non-functional needs:** identity/auth model ([PLATFORM §5](../../../rules/PLATFORM.md)),
  scale (recall [§3](../../../rules/PLATFORM.md): hundreds not millions — modular monolith),
  external integrations, events between domains.
- **Explicit non-goals.**
Ask only what you cannot reasonably infer; propose sensible defaults instead of interrogating.

## Step 1 — shape the domain model
- List **bounded contexts (domains)** and their **entities**. A domain depends on another
  only through its public **protocol** or an **event**, never by importing internals
  (rules 41–44).
- Define the **API contract surface** — endpoints per domain, designed from the domain,
  never from a screen (rules 1–2).
- Name the **identity model**, the main **error cases**, and where **events** integrate domains.

## Step 2 — decompose into phases
Sequence so each phase is **shippable and rules-compliant**. Canonical order:

1. **Foundation** — project skeleton ([BACKEND_STRUCTURE](../../../rules/BACKEND_STRUCTURE.md)):
   `core`, settings, the dishka container, `health`, CI, and the
   [GIT_AND_DELIVERY](../../../rules/GIT_AND_DELIVERY.md) workflow (trunk-based, PR, release).
2. **Identity & auth** ([PRODUCTION §1–2](../../../rules/BACKEND_RULES_PRODUCTION.md)) — if the product needs it.
3. **Domains** — one bounded context at a time (`new-domain`), highest-value slice first;
   add endpoints as vertical slices (`new-endpoint`).
4. **Cross-domain events** (`new-event`) where domains integrate.
5. **Frontend surfaces** (`new-frontend-section`) once the contract is stable — types come
   from the OpenAPI schema ([FRONTEND_STRUCTURE](../../../rules/FRONTEND_STRUCTURE.md)).
6. **Production hardening** ([PRODUCTION](../../../rules/BACKEND_RULES_PRODUCTION.md) critical):
   observability, idempotency, rate limits, deploy.

Each phase names: the **build skill** to run, the **rules sections** in scope, and the
**done criteria** (tests + typecheck + `rules-review` green, OpenAPI diff clean).

## Step 3 — write the plan to `plans/`
- **`plans/PLAN.md`** — the phased plan: phases, slices, order, per-step build skill, done criteria.
- **`plans/DECISIONS.md`** — key architecture decisions with rationale (ADR-style).
- **`plans/design/<topic>.md`** — deeper design per domain/topic as needed.

Product-specific rules and doctrines live in **`plans/`**, never in `rules/` — the ruleset
stays product-agnostic (see [rules/README.md](../../../rules/README.md)).

## Step 4 — hand off to build
- **Confirm the plan** with the user before writing code.
- Execute **phase by phase**: the matching build skill per slice; run `rules-review`
  before each merge; follow `GIT_AND_DELIVERY` per slice (branch → PR → squash).
- Keep `plans/PLAN.md` updated as phases complete (check items off, note deviations).

## Verification — a good plan
- [ ] Every user story maps to a slice inside a phase.
- [ ] Domains are isolated; integrations go through a protocol or an event (rules 41–44).
- [ ] Contract-first: endpoints are designed from the domain, not the screen (rules 1–2).
- [ ] Each phase has explicit done criteria and a named build skill.
- [ ] Non-functionals (auth, observability, deploy) are phased in, not forgotten.
- [ ] The plan lives in `plans/`; `rules/` stays product-agnostic.
