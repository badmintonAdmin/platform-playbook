# Platform Playbook

An engineering ruleset for building products on a shared platform: **how** to build
backend and frontend in a way that is high-quality, extensible, and maintainable. It is
grounded in **clean architecture** and **DDD**, with a single API treated as a product
serving multiple clients (web, mobile, integrations).

The rules are **reusable across products**: they describe *how to build*, not *what the
product is*. Concrete subject areas, integrations, and domain doctrines belong to a
separate product layer built on top of this playbook.

> 🇷🇺 **Русская версия:** [`ru/`](ru/) — the original Russian edition (rules + skills).
> This root is the English edition.

---

## What's inside

- **[`rules/`](rules/)** — the ruleset: cross-cutting platform principles, backend and
  frontend rules, the reference structure, and a single numbered checklist.
- **[`.claude/skills/`](.claude/skills/)** — companion [Claude Code](https://claude.com/claude-code)
  skills that operationalize the ruleset (create a domain, add an endpoint, design an
  event, add a frontend section, review against the rules, keep the rules in sync).

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
4. **[BACKEND_RULES_PRODUCTION.md](rules/BACKEND_RULES_PRODUCTION.md)** — production
   concerns: security (authn/authz, OWASP), observability, fault tolerance, deployment,
   data reliability. Read before going to prod.
5. **[RULES_CHECKLIST.md](rules/RULES_CHECKLIST.md)** — the entire ruleset as one
   numbered checklist (115 rules, one line each) for code review and PRs.

### Priority on conflict

```
PLATFORM.md  >  BACKEND_RULES.md / FRONTEND_STRUCTURE.md  >  *_STRUCTURE / *_PRODUCTION
```

If a specific rule contradicts a platform principle, the **principle wins** (or the
specific rule is fixed). The product layer may only tighten platform invariants for its
subject area, never weaken them.

## Companion skills

The rules are operationalized as skills in [`.claude/skills/`](.claude/skills/):

| Skill | What it does |
| --- | --- |
| `new-domain` | Scaffold a new backend domain (bounded context) across all layers. |
| `new-endpoint` | Add an endpoint to an existing domain as a vertical slice. |
| `new-event` | Design a domain event: contract, Transactional Outbox, idempotent consumer. |
| `new-frontend-section` | Add a page/section/surface to the web client. |
| `rules-review` | Audit a diff/PR against the 115-rule checklist. |
| `rules-sync` | Keep the checklist, cross-references, and README in sync after rule changes. |

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
