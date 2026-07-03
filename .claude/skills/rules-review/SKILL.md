---
name: rules-review
description: Check a diff / PR / files against the 128 platform rules in RULES_CHECKLIST.md and produce a report of violations with rule numbers. Use when the user asks to check code against the platform rules ("check against the rules", "checklist audit", "rules review/audit", before merging significant changes).
---

# Platform Rules Audit

Source of truth: [rules/RULES_CHECKLIST.md](../../../rules/RULES_CHECKLIST.md) (128
rules). When a checklist line is ambiguous, open the full document
(PLATFORM / BACKEND_RULES / BACKEND_STRUCTURE / PRODUCTION / FRONTEND_STRUCTURE);
it takes precedence.

This is a **rules-compliance audit**, not a general bug hunt. Don't turn it into a
code review: flag logic errors in a separate "outside the checklist" section, briefly.

## Step 1 — determine the scope
- No arguments — the current diff (`git diff` + staged; if empty — the last commit).
- A PR/branch/files given — take those.
- List the affected areas: backend domains, core, migrations, events, frontend.

## Step 2 — select the applicable sections
Don't run all 128 rules against every file — take the relevant sections:

| In the diff | Checklist sections |
|---|---|
| `apps/*` (backend) | Layers (17–32), DI (33–39), Domains (40–50), Errors (66–69), Security (80–88) |
| `models` / `migrations` | DB models (51–61), Migrations (62–65) |
| `events.py` / consumers / jobs | Events (70–76), Background work (77–79) |
| routers / schemas (contract) | Platform/API (1–16) |
| frontend | Frontend (103–115), Platform/API (5–11, 14) |
| infrastructure / CI / Docker | Operations (89–97), Security (86–87), Tests (98–102) |
| branches / PRs / release / deploy | Git and delivery (116–128) |

## Step 3 — verify and reach a verdict
For each candidate violation — **confirm against the code** (read the context, don't
judge from a single diff line). Discard false positives.

## Step 4 — report
Format — in descending order of severity:

```
## Violations
- **№25** (ORM above the repository) — `apps/orders/services.py:42`:
  the service returns an Order model. Fix: return OrderReadSchema.

## Debatable / judgment call
- №30 … (why it's debatable)

## Compliant
Sections checked: Layers, DI, DB models — no violations.
```

- A violation = number + gist + `file:line` + a short fix.
- Fix nothing without a request; offer a separate `--fix` pass.
- If there's no diff and no scope is given — ask what to check rather than scanning the entire repository.
