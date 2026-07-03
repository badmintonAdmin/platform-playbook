# Platform Rules

The platform's engineering ruleset: how to build backend and frontend in a way that is
high-quality, extensible, and manageable. The foundation is **clean architecture** and
**DDD**, with a single API as a product serving multiple clients (web, mobile,
integrations).

The rules are **reusable across products**: they describe *how to build*, not *what the
product is*. Specific domains, integrations, and domain doctrines are a separate product
layer on top of these rules.

---

## Document Map

### Quick Check
- **[RULES_CHECKLIST.md](RULES_CHECKLIST.md)** — the entire ruleset as a single numbered
  checklist (115 rules, one line = one rule) for reviews and PRs. In case of conflict,
  the full document wins.

### Platform Layer (above everything, takes priority on conflict)
- **[PLATFORM.md](PLATFORM.md)** — cross-cutting principles: API-first / multi-client,
  the API contract as the single seam, casing, a unified error format, versioning,
  an identity model for all clients, configuration and environments, observability.

### Backend
- **[BACKEND_RULES.md](BACKEND_RULES.md)** — the laws: stack, clean architecture, layers,
  DI (dishka), protocols vs. implementations, repositories, models, migrations, errors,
  versioning, quality.
- **[BACKEND_STRUCTURE.md](BACKEND_STRUCTURE.md)** — the reference structure to copy:
  the domain tree, layer skeletons, naming, and **how the dishka dependency graph is
  assembled and woven into the router**.
- **[BACKEND_RULES_PRODUCTION.md](BACKEND_RULES_PRODUCTION.md)** — production:
  security (authn/authz, OWASP), observability, fault tolerance, deployment,
  data reliability.

### Frontend
- **[FRONTEND_STRUCTURE.md](FRONTEND_STRUCTURE.md)** — the structure and baseline rules
  for the web client (Next.js App Router) as one of the API's clients.

---

## Priority on Conflict

```
PLATFORM.md  >  BACKEND_RULES.md / FRONTEND_STRUCTURE.md  >  *_STRUCTURE / PRODUCTION
```

If a specific rule contradicts a platform principle, **the principle wins** (or the
specific rule is corrected). The product layer cannot weaken the platform's
security/architecture invariants, only tighten them for its own domain.

## Reading Order (for a new engineer)

1. [PLATFORM.md](PLATFORM.md) — how the platform is built and where its boundaries lie.
2. [BACKEND_RULES.md](BACKEND_RULES.md) → [BACKEND_STRUCTURE.md](BACKEND_STRUCTURE.md)
   — the architecture and how to reproduce it in code.
3. [FRONTEND_STRUCTURE.md](FRONTEND_STRUCTURE.md) — the client side.
4. [BACKEND_RULES_PRODUCTION.md](BACKEND_RULES_PRODUCTION.md) — before going to production.

---

## Companion Skills

The rules are operationalized through skills in [.claude/skills/](../.claude/skills/) —
standard procedures that execute the ruleset in practice: `new-domain`, `new-endpoint`,
`new-event`, `new-frontend-section`, `rules-review`, `rules-sync` (the map is in the root
[CLAUDE.md](../CLAUDE.md)). Changing the rules entails updating the skills — this is the
`rules-sync` step.

## Status and Notes

- The ruleset evolves **iteratively**: first the base and cleanup, then in-depth passes
  through the sections.
- Product rules and doctrines do not belong in this ruleset — their place is in the
  project's `plans/` (see [plans/DECISIONS.md](../plans/DECISIONS.md) and `plans/design/`).
