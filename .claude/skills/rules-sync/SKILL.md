---
name: rules-sync
description: Synchronize the ruleset after changes — the checklist matches the full documents, numbers and cross-references have not drifted apart, and the README is up to date. Use after editing anything in rules/ ("update the checklist", "synchronize the rules", "rules sync"), or when asked to add/change a platform rule.
---

# Ruleset synchronization

Ruleset: `rules/` — full documents + [RULES_CHECKLIST.md](../../../rules/RULES_CHECKLIST.md)
(summary) + [README.md](../../../rules/README.md) (index). Invariant: **the checklist is a
projection of the full documents**; any divergence is a documentation bug.

## When a rule is added/changed
1. **Full document first** (PLATFORM / BACKEND_RULES / BACKEND_STRUCTURE /
   PRODUCTION / FRONTEND_STRUCTURE) — the rule with its rationale ("why"), in the correct
   section, following the hierarchy (platform-level goes in PLATFORM, do not duplicate downward).
2. **Then the checklist** — one line per rule, in the corresponding section.
   Renumbering is acceptable (references to numbers live only inside the repo — update them:
   `grep -rn "rule" .claude/skills/ rules/` for the affected numbers).
3. **README** — update the rule count in the checklist description; for a new document,
   add it to the map and the reading order.
4. **Skills** — if a rule changes a procedure (a new step, a new check), update the
   corresponding skill in `.claude/skills/`.

## Consistency checks (always run)
- [ ] Every checklist rule has a source in a full document (and vice versa — new
      "hard" rules in the documents are reflected in the checklist).
- [ ] Numbering is continuous with no gaps; the count in the README matches.
- [ ] Cross-references of the form "§N" and "rule N" point to existing sections
      (when sections are inserted into the middle of a document the numbers shift — verify).
- [ ] No contradictions between documents: on a conflict, the higher one in the hierarchy wins
      (README → "Priority on conflict"); a contradiction must be fixed, not left in place.
- [ ] Links between files (`[…](FILE.md)`) are not broken.

## Commit
`docs(rules): <what changed>` — Conventional Commits; rule edits and checklist
synchronization go in a single commit, so the history never contains out-of-sync states.
