# Platform Playbook

**A starter kit for building a production-grade full-stack product from an idea — fast,
and by the rules.** Clone it, point your AI coding agent (e.g.
[Claude Code](https://claude.com/claude-code)) at it, and describe what you want to build:
the `plan` skill turns your idea into a phased delivery plan, and the build skills execute
it slice by slice — so a solo full-stack developer can go from idea to a well-architected
product almost in a single prompt.

Under the hood it is an engineering **ruleset**: *how* to build backend and frontend so the
result is high-quality, extensible, and maintainable — grounded in **clean architecture**
and **DDD**, with a single API treated as a product serving multiple clients (web, mobile,
integrations). The rules are **reusable across products**: they describe *how to build*,
not *what the product is*; the product itself is a separate layer built on top.

> **Stack:** a **FastAPI (Python) backend + Next.js (TypeScript) frontend**. Many rules
> encode stack-specific practice (dishka DI, SQLAlchemy/Alembic, Pydantic, the App Router);
> on another stack the principles may hold but the specifics will not.

> 🇷🇺 **Русская версия:** [`ru/`](ru/) — the original Russian edition (rules + skills).
> This root is the English edition. Pick the language you work in and delete the other.

---

## What's inside

- **[`rules/`](rules/)** — the ruleset: cross-cutting platform principles, backend and
  frontend rules, the delivery workflow, the reference structure, and a single numbered
  checklist.
- **[`.claude/skills/`](.claude/skills/)** — companion [Claude Code](https://claude.com/claude-code)
  skills that operationalize the ruleset: `plan` (idea → phased delivery plan), then
  create a domain, add an endpoint, design an event, add a frontend section, review
  against the rules, keep the rules in sync.

## Getting started (as a starter)

1. **Clone** this repo as the seed of your new project.
2. **Pick a language edition** — keep `rules/` + `.claude/skills/` (English) *or* the
   `ru/` edition, and delete the other. Both carry the same ruleset and skills.
3. **Open it with [Claude Code](https://claude.com/claude-code)** (the skills live in
   `.claude/skills/` and load automatically).
4. **Describe your idea and run `plan`** — you get a phased plan in `plans/` that honors
   the whole ruleset.
5. **Build phase by phase** with the build skills; `rules-review` gates each merge and
   `GIT_AND_DELIVERY` governs how it ships.

## Document map

Read in this order (see [`rules/README.md`](rules/README.md) for the full map):

1. **[PLATFORM.md](rules/PLATFORM.md)** — cross-cutting principles: API-first /
   multi-client, the API contract as the single seam, casing, the unified error format,
   versioning, one identity model for all clients, configuration, observability.
   *(Highest priority — wins on conflict.)*
2. **[BACKEND_RULES.md](rules/BACKEND_RULES.md)** → **[BACKEND_STRUCTURE.md](rules/BACKEND_STRUCTURE.md)**
   — the backend laws (stack, clean architecture, layers, DI via dishka, protocols vs
   implementations, repositories, models, migrations, errors) and the canonical
   reference structure to copy for a new domain.
3. **[FRONTEND_STRUCTURE.md](rules/FRONTEND_STRUCTURE.md)** — the web client (Next.js App
   Router) as one of the API's clients.
4. **[GIT_AND_DELIVERY.md](rules/GIT_AND_DELIVERY.md)** — how code ships: trunk-based
   branching, pull requests, squash-merge, automatic release, environment promotion,
   zero-downtime deployment and rollback (with non-normative tooling recommendations).
5. **[BACKEND_RULES_PRODUCTION.md](rules/BACKEND_RULES_PRODUCTION.md)** — production
   concerns: security (authn/authz, OWASP), observability, fault tolerance, deployment,
   data reliability. Read before going to prod.
6. **[RULES_CHECKLIST.md](rules/RULES_CHECKLIST.md)** — the entire ruleset as one
   numbered checklist (128 rules, one line each) for code review and PRs.

### Priority on conflict

```
PLATFORM.md / GIT_AND_DELIVERY.md  >  BACKEND_RULES.md / FRONTEND_STRUCTURE.md  >  *_STRUCTURE / *_PRODUCTION
```

If a specific rule contradicts a platform principle, the **principle wins** (or the
specific rule is fixed). The product layer may only tighten platform invariants for its
subject area, never weaken them.

## Companion skills

The rules are operationalized as skills in [`.claude/skills/`](.claude/skills/):

| Skill | What it does |
| --- | --- |
| `plan` | Turn an idea into a phased delivery plan in `plans/`, then hand off to the build skills. |
| `new-domain` | Scaffold a new backend domain (bounded context) across all layers. |
| `new-endpoint` | Add an endpoint to an existing domain as a vertical slice. |
| `new-event` | Design a domain event: contract, Transactional Outbox, idempotent consumer. |
| `new-frontend-section` | Add a page/section/surface to the web client. |
| `rules-review` | Audit a diff/PR against the 128-rule checklist. |
| `rules-sync` | Keep the checklist, cross-references, and README in sync after rule changes. |

The usual flow is **`plan` → build skills per slice → `rules-review` before each merge**.
Changing the rules implies updating the skills — that is the `rules-sync` step.

## Repository layout

```
README.md                     ← this file (English edition)
rules/                        ← ruleset (English)
.claude/skills/               ← companion skills (English)
ru/
  rules/                      ← ruleset (Russian, original)
  .claude/skills/             ← companion skills (Russian, original)
```

## License

Released under the [MIT License](LICENSE).
