# Git and delivery

> Platform-level, **cross-cutting** doc: it applies to backend and frontend alike. It
> describes **how code moves from a branch to production** — branching, pull requests,
> merge, release, and deployment. The *commit convention* (Conventional Commits, SemVer,
> semantic-release) lives in [BACKEND_RULES.md §14](BACKEND_RULES.md); this document owns
> the **workflow** around it. It sits at the platform layer next to
> [PLATFORM.md](PLATFORM.md) and **wins on conflict** with lower documents.
>
> **Stack scope:** like the rest of this playbook, the concrete guidance assumes a
> **FastAPI (Python) backend + Next.js (TypeScript) frontend** shipped as Docker images.
> The *principles* (trunk-based, build-once-promote, expand→contract migrations,
> zero-downtime rollout, rollback by redeploy) are stack-agnostic; the *tooling notes*
> at the end are not.

---

## 1. Branching — trunk-based

- **`main` is the trunk**: it is always releasable and always deployable. Every commit on
  `main` is a candidate for production.
- **Short-lived branches only.** Branch off `main`, do one thing, merge back within a day
  or two. No long-lived `develop`/`release`/`feature` branches that drift from the trunk.
- **Branch naming:** `type/short-slug`, where `type` matches the Conventional Commit type —
  `feat/orders-checkout`, `fix/session-expiry`, `chore/bump-uv`, `docs/api-guide`.
- **One branch = one vertical slice** (see [BACKEND_RULES.md §4](BACKEND_RULES.md), a user
  story end to end), not a grab-bag of unrelated changes.

## 2. Pull requests

- **Every change reaches `main` through a PR.** No direct pushes to `main` — it is a
  protected branch.
- **A PR is small and single-purpose.** It should be reviewable in one sitting; if it is
  hard to review, split it. Prefer stacked PRs over one giant branch.
- **CI must be green to merge:** lint, type-check, tests, and the **OpenAPI diff** all run
  on the PR. A breaking contract change is caught here, not in production
  ([PLATFORM.md §2](PLATFORM.md), rule 7).
- **At least one review** before merge; the author does not approve their own PR.
- The PR description states *what* and *why*; the *how* is in the diff.

## 3. Merge and history

- **Squash-merge only.** A PR collapses into **one Conventional Commit** on `main`; that
  squash-commit title is the release-driving message (see §4).
- **Linear history:** no merge commits. Rebase a stale branch onto `main` before merging;
  never merge `main` back into a feature branch.
- The commit body carries `BREAKING CHANGE:`/`!` when the contract breaks
  ([BACKEND_RULES.md §14](BACKEND_RULES.md)).

## 4. Release

- **Automatic, from the trunk.** On push to `main`, **semantic-release** analyzes the
  Conventional Commits, computes the next **SemVer** version, tags it, and builds and
  publishes an **immutable image** ([BACKEND_RULES.md §14/§17](BACKEND_RULES.md)).
- **The version is derived from commits, never set by hand.** Meaningful commit messages
  are therefore mandatory — the release depends on them.
- The version-bump commit carries `[skip ci]` so it does not trigger a second run.

## 5. Environments and promotion

- **Environments flow `local → staging → production`** ([PLATFORM.md §6](PLATFORM.md)).
- **Build once, promote the same artifact.** The image built on merge is the exact image
  that runs in staging and then in production — **never rebuilt per environment**. Only
  **config and secrets** differ between environments, injected from the environment
  (12-factor, [PLATFORM.md §6](PLATFORM.md), secrets from a manager per
  [PRODUCTION §14](BACKEND_RULES_PRODUCTION.md)).
- **Staging deploys automatically** on a new release. **Production is gated** by an
  explicit manual approval — a human promotes the already-tested artifact; production is
  never an unattended auto-push.

## 6. Deployment

- **Migrations run as a separate pre-deploy step** (a job / init step), **before** the new
  code takes traffic, and are **backward-compatible (expand→contract)** so the old code
  keeps working against the new schema ([BACKEND_RULES.md §9](BACKEND_RULES.md)).
- **Zero-downtime rolling rollout**, gated by the **readiness** probe: new instances only
  receive traffic once healthy, old instances drain in-flight work on `SIGTERM`
  (graceful shutdown, [PRODUCTION §4](BACKEND_RULES_PRODUCTION.md)). Old and new instances
  briefly coexist — hence the compatibility rule above.
- **The runtime is stateless and configured from the environment**; logs go to stdout, not
  to a file ([PRODUCTION §5](BACKEND_RULES_PRODUCTION.md), [PLATFORM.md §6](PLATFORM.md)).
- The image is **non-root, multi-stage, pinned by digest**
  ([PRODUCTION §5](BACKEND_RULES_PRODUCTION.md), rule 93).

## 7. Rollback

- **Rollback = redeploy the previous image.** Because migrations are **expand→contract**,
  a code rollback **never requires a schema rollback** — the previous release's code is
  already compatible with the current schema.
- Destructive schema changes (the "contract" step: dropping a column/table) ship **only
  after** the code that stopped using them is safely in production for at least one
  release — so a rollback within that window stays safe.

## 8. Traceability of delivery

- **Every deploy is traceable to a commit and a version tag.** The running artifact is
  identifiable (image digest / a version surfaced by the app), and correlates with the
  release that produced it.
- Deploys are recorded (who promoted what, when) so an incident can be tied to a change.

---

## Recommended tooling (informational, not rules)

The rules above are deliberately tool-agnostic. Concrete tooling is a **recommendation**,
not a platform rule — choose by scale, and remember the platform targets **hundreds of
users, simplicity over horizontal scale, a modular monolith by default**
([PLATFORM.md §3](PLATFORM.md)). For a FastAPI + Next.js product that posture usually means
a lightweight PaaS over a full orchestrator.

- **Recommended default — [Dokploy](https://dokploy.com):** a self-hosted, open-source
  PaaS (a Heroku/Vercel-style layer over Docker). It deploys the built images, manages
  environments and secrets, terminates TLS, and does zero-downtime rollouts and one-click
  rollbacks — a good fit for a FastAPI backend + Next.js frontend on a single VM, and
  aligned with the simplicity-first posture.
- **Similar self-hosted PaaS:** **Coolify** — comparable model if you prefer its UX.
- **Docker-native deploy tool:** **Kamal** — zero-downtime rolling deploys over SSH,
  no PaaS layer, if you want to stay close to plain Docker.
- **Managed PaaS (no server to run):** **Render**, **Railway**, **Fly.io** — trade some
  control for zero ops.
- **Scale-out:** **Kubernetes** ([PRODUCTION §5](BACKEND_RULES_PRODUCTION.md) — probes,
  resources, manifests) once a single-host PaaS is genuinely outgrown. It is the heavier
  option, **not the starting point**.

Whatever the tool, it must honor the principles above: **one immutable artifact promoted
across environments, config from the environment, pre-deploy expand→contract migrations,
readiness-gated zero-downtime rollout, and rollback by redeploying the previous image.**

---

## Summary — do / don't

**Do**
- Trunk-based: short branches off `main`, squash-merge one Conventional Commit, linear history.
- Protect `main`: PR + green CI (incl. OpenAPI diff) + review; release automatically via semantic-release.
- Build once, promote the same image; config from the environment; production gated by a human.
- Migrate expand→contract before rollout; roll out zero-downtime behind readiness; roll back by redeploy.

**Don't**
- Push straight to `main`, keep long-lived branches, or merge `main` into a feature branch.
- Rebuild the image per environment, or bake config/secrets into the image.
- Ship a destructive migration in the same release as the code that stops using the column.
- Hardcode a specific deploy tool into the *rules* — keep tooling a recommendation.
