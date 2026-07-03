# Production best-practices (supplement to BACKEND_RULES.md)

> Supplement to [BACKEND_RULES.md](BACKEND_RULES.md). The base rules cover
> **code architecture** (clean architecture, layers, repositories, models, dishka, errors).
> This document covers **production operations**: security, observability,
> fault tolerance, operational work with data, and deployment. Cross-cutting platform
> contracts (unified error format, API versioning, identity model for all
> clients) are in [PLATFORM.md](PLATFORM.md).
>
> References of the form «§N» point to sections of the base file. The new rules build on
> the already-adopted stack: identity/sessions/locks — via **dishka providers and scope**,
> authz — in the domain layer, outbox — on top of the existing **Unit of Work**,
> stampede lock — on top of the distributed Lock from core.
>
> Priorities: **Critical** (cannot go to production without it) → **Important** (mature service) →
> **Recommended** (raises maturity, context-dependent).

---

# Part I. Critical

## 1. Authentication and authorization

The base rules include `PermissionDenied → 403` (§11) and `SECRET_KEY` (§12), but no
rules for **where authn/authz live in clean architecture**. This is a critical gap.

### AuthN (authentication)
- **A dedicated `apps/auth/` domain** following the reference domain structure (§3).
- **JWT access token** — short TTL (5–15 min) + **refresh token with rotation**.
  For revocation, store `jti` (or a refresh-token whitelist) in Redis/DB — without this
  a token cannot be invalidated before it expires.
- **Password hashing — Argon2id** (`argon2-cffi` or `pwdlib`), parameters per OWASP.
  Never sha/pbkdf2/md5 by hand. (`passlib` is currently poorly maintained.)
- **OAuth2 / OIDC** — `authlib` (external providers, single sign-on).
- JWT library — `pyjwt` or `joserfc`.

### AuthZ (authorization)
- **Permission checks — in the UseCase/Service layer, NOT in the router.** The router only extracts
  identity from the token; the decision "is this allowed" is a domain rule. `PermissionDenied`
  (§11) is raised from the domain (see §6 of the base rules — business logic in UseCase).
- **RBAC** (roles) as the baseline; **ABAC**/policy (`oso`/`casbin`) — for complex rules.
- **BOLA (object-level)** — verify **ownership of the specific resource** in every
  UseCase, not just the fact of authentication (see §2 — this is OWASP API #1).

### Where to get the current user
- **`CurrentUser` — via a dishka REQUEST-scope provider** (§10), not a global
  `contextvars`/singleton. The provider retrieves identity from the request token; UseCase/Service
  receive `CurrentUser` as an ordinary dependency.

```python
# dishka provider for identity (REQUEST scope)
@provide(scope=Scope.REQUEST)
def current_user(self, request: Request, auth: AuthServiceProtocol) -> CurrentUser:
    token = extract_bearer(request)
    return auth.identify(token)          # raises AuthenticationError if invalid
```

**Why it's critical:** "bolting on" authn/authz later means rewriting the access layers and
leaking permission checks across controllers.

---

## 2. API security (OWASP API Security Top 10)

The base rules mention only `CORS (allow_origins)` (§12). There is no protection against brute force,
no security headers, no protection against typical API attacks.

| Threat | Rule |
|---|---|
| **Brute force / DoS** | **Rate limiting** — `slowapi` or Redis-based middleware (sliding window / token bucket). Strict limits on `login`/`refresh`/`forgot-password`. |
| **Headers** | **Security headers** (the `secure` library or middleware): `Strict-Transport-Security`, `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Content-Security-Policy`, `Referrer-Policy`. Remove the `Server` header. |
| **CORS** | Explicit origin allowlist. **Forbidden**: `allow_origins=["*"]` together with `allow_credentials=True`. |
| **CSRF** | For web clients on cookie sessions: `SameSite=Lax/Strict` + a CSRF token on mutating requests (see PLATFORM §5). Does not apply to bearer clients. |
| **BOLA (#1)** | Verify resource ownership in every UseCase (see §1). Authentication ≠ authorization on an object. |
| **BOPLA / mass assignment** | Strict Pydantic schemas: `model_config = ConfigDict(extra="forbid")`. Separate Create/Update schemas (base §7) — do not accept `id`/`role`/`is_admin` from the client. |
| **Unrestricted consumption** | Limits on request body size, pagination depth, and the number of elements in a bulk operation. |

**Why it's critical:** BOLA is the leading cause of API leaks; mass assignment through
unlocked Pydantic schemas is a silent privilege-escalation hole.

---

## 3. Observability: structured logs and correlation id

The base rules initialize logging (§3), but without structure or request tracing.

- **Structured JSON logs** — `structlog` (preferred) or the stdlib `logging`
  with a JSON formatter. No `print` and no unstructured strings in production.
- **Correlation / Request ID** — middleware (`asgi-correlation-id`) that
  accepts/generates `X-Request-ID`, puts it in `contextvars`, and injects it
  into **every log line and every error**. Propagate it further into outgoing HTTP/broker calls.
- **No PII/secrets in logs** — an explicit list of redacted fields
  (`password`, `token`, `authorization`, and `email` if needed).
- Logs — to **stdout** (12-factor), not to a file; the aggregator collects from there (see §5).

**Why it's critical:** without request_id and structured logs, a production incident is uninvestigable —
the lines of a single request cannot be correlated in a distributed system.

---

## 4. Graceful shutdown and timeouts

In the base rules, `lifespan` is mentioned only for closing the dishka container (§10).

### Graceful shutdown
- Proper handling of **SIGTERM**: wait for in-flight requests to complete, then
  in `lifespan` shutdown close the DB/Redis/broker pools, the **dishka container**
  (`await container.close()`), stop APScheduler and FastStream consumers.
- In k8s: `terminationGracePeriodSeconds` ≥ the duration of long requests + a `preStop` hook
  (a short pause so the LB can remove the pod from rotation).

### Timeouts — on everything
- **The HTTPX client always with an explicit `timeout=`** (connect/read/write/pool). The default
  "no timeout" is forbidden.
- **DB** — `statement_timeout` on the session/pool (protection against hung queries).
- **External calls** wrapped in `asyncio.timeout(...)`.

**Why it's critical:** a release without graceful shutdown = severed requests and lost
transactions on every rollout. A request without a timeout drains the pool and takes down the service.

---

## 5. Production deployment (image and k8s)

The base rules describe a Dockerfile and docker-compose for infrastructure (§16/§17),
but not a production image and not the manifests.

### Dockerfile
- **Multi-stage**: builder (`uv sync --frozen --no-dev`) → slim runtime image.
- **Non-root**: create a user, `USER appuser`. Read-only rootfs where possible.
- The base image is **pinned by digest** (`python:3.13-slim@sha256:...`).
- Migrations go into the image (base §3); dev artifacts do not (`.dockerignore`).

### Kubernetes
- **Separate probes** (not a single "200 healthcheck"):
  - **liveness** — shallow: the process is alive, **without checking dependencies** (otherwise a DB flap
    restarts the pods).
  - **readiness** — deep: DB/Redis/broker are reachable (see §16).
  - **startup** — for slow startup (migrations/warm-up).
- **`resources.requests/limits`** (CPU/mem) — mandatory.
- **12-factor**: stateless processes, config from env (base §12), logs to stdout
  (not to a `logging.dev` file in production).

**Why it's critical:** a root container and a single healthcheck are the classic causes of
privilege escalation and cascading restarts of the entire deployment when the DB blinks.

---

# Part II. Important

## 6. Metrics and tracing

Core mentions Prometheus (base §13) and an optional `SENTRY_DSN` (§12), but without rules.

- **Metrics** — `prometheus-fastapi-instrumentator`: RED (Rate/Errors/Duration) +
  a `/metrics` endpoint. Business metrics — in a separate registry.
- **Tracing** — **OpenTelemetry** (`opentelemetry-instrumentation-fastapi`,
  `-sqlalchemy`, `-httpx`), with spans tied to the same correlation id (§3).
- **Sentry** — make it a standard (not an option), tied to `request_id` and
  the `release` version from semantic-release (base §14).

**Why it's important:** without metrics there is no SLO/alerting; without tracing it is unclear where
the latency is in the chain of services.

---

## 7. Soft delete and audit

Base §18 mentions `deleted_at` only in the checklist; it is absent from §8 (models), and there are no
rules for **how to live with it**.

- **`SoftDeleteMixin (deleted_at: datetime | None)`** — soft delete.
- **The repository filters `deleted_at IS NULL` by default**; the `with_deleted=True` flag
  disables the filter explicitly.
- **Partial unique index**: `UNIQUE (...) WHERE deleted_at IS NULL` — otherwise uniqueness
  breaks after a soft delete (you cannot recreate a "deleted" record).
- **Audit** — `created_by` / `updated_by` (UUID from `CurrentUser`, §1), populated in
  the Service/UoW layer. For the full change history — a history table / triggers /
  `sqlalchemy-continuum`.
- **Soft delete ≠ erasure (PII/GDPR).** On a data-deletion request or when the PII retention
  period expires, data **is physically deleted or anonymized** — including the audit,
  history, and related records. Retention periods are defined per entity;
  "soft-deleted and kept forever" for personal data is a violation.

**Why it's important:** "forgot to filter out deleted records" and "uniqueness broke after
soft delete" are widespread bugs; a "who changed it" audit is needed for compliance.

---

## 8. Concurrency: locks and isolation

Base §8 does not mention concurrent access. The generic CRUD `update` (§7) without a version =
lost update under load.

- **Optimistic locking** — a `version_id` column +
  `__mapper_args__ = {"version_id_col": version}` (SQLAlchemy adds
  `WHERE version = :v` itself and raises `StaleDataError`). The default for concurrent updates.
- **Pessimistic** — `SELECT ... FOR UPDATE` for critical spots (balances, counters).
- **Isolation level** — default `READ COMMITTED`; raise to `REPEATABLE READ`/
  `SERIALIZABLE` selectively where protection against phantom/non-repeatable reads is needed.

**Why it's important:** without this, two parallel updates silently overwrite each other.

---

## 9. Idempotency of mutations

Base §7 provides `upsert ON CONFLICT`, but there is no idempotency at the API/event level.

- **`Idempotency-Key`** — a header for POST (payments, order creation). Middleware
  stores `key → result` in Redis with a TTL; on a retry it returns the stored response.
- **Idempotent event consumers** — deduplication by `message_id` (inbox table, §11).

**Why it's important:** client and broker retries are inevitable; without idempotency you get double
charges and duplicates.

---

## 10. API design: versioning, pagination, filtering

Base §7 provides pagination in the repository and camelCase, but not an API standard.

- **Versioning** — path prefix **`/api/v1/`**. A breaking-changes rule for the API
  (by analogy with SemVer from §14): an incompatible change → a new path version.
- **Keyset/cursor pagination** as the default for large lists — offset degrades on
  large tables and "jumps" on inserts. Response `{items, next_cursor}`.
  `fastapi-pagination` or a custom schema.
- **Filtering/sorting** — a unified query-parameter contract (`fastapi-filter`),
  not ad hoc on each endpoint.

**Why it's important:** offset pagination and the absence of versioning are expensive tech debt
after an API is published.

---

## 11. Outbox pattern and reliable messaging

The stack has FastStream (base §1), but no rules for reliable publishing. Writing to the DB +
publishing to the broker = the classic **dual-write** problem (the event is lost on a crash
between commit and publish).

- **Transactional Outbox** — the event is written to an `outbox` table in **the same transaction**
  as the domain change (using the existing **UoW**, base §6). A separate
  relay/poller reads `outbox` and publishes to the broker, marking what has been sent.
- **Idempotent consumers** + an **inbox** table for deduplication (§9).
- **DLQ** (dead letter queue) for "poison" messages + retry with backoff (§15).

**Why it's important:** without the outbox, events are lost → state drift between services.

---

## 12. Caching

Redis is in the infrastructure (§16) and a cache repository is in core (§13), but there are no patterns.

- **Cache-aside** as the standard: read → cache → miss → db → populate the cache;
  write → invalidate the key.
- **A mandatory TTL** on everything — no eternal keys.
- **Stampede protection** — when a popular key expires, don't let everyone into the DB:
  a distributed **Lock from core** (§13) on the rebuild, or an early recompute.
- **Invalidation on domain events**, not "by eye".

**Why it's important:** a cache without TTL/invalidation serves stale data; a cache stampede
takes down the DB when a hot key expires.

---

## 13. Testing

Base §15 provides pytest + an `alembic upgrade head` fixture (slow, on every test).

- **testcontainers-python** — integration tests on a real Postgres/Redis in
  an isolated container (instead of a shared dev DB).
- **Data factories** — `polyfactory` (native for Pydantic/SQLAlchemy) instead of hand-written
  fixtures.
- **The test pyramid explicitly**: **many unit tests** (the domain without I/O is cheap — Protocol + in-memory
  repositories, base §5/§7), **fewer integration tests**, **a minimum of e2e**.
- **Contract tests** — `schemathesis` (fuzzing against OpenAPI); `pact` for
  consumer-driven testing in microservices.
- **Rollback to a savepoint** around a test instead of a full `upgrade head` — many times faster.

**Why it's important:** slow/brittle tests don't get run. Clean architecture already provides
cheap unit tests — this needs to be codified as a rule.

---

## 14. Secrets management and SAST/SCA

Base §12 — secrets only via `.env`; §17 CI — lint + tests without security scans.

- **Production secrets** — from a vault/secrets manager (HashiCorp Vault, AWS/GCP Secrets Manager,
  k8s Secrets), not from an `.env` file on disk. `.env` is for local development only.
- **Pre-commit**: `detect-secrets` / `gitleaks` — prevent a secret from being committed.
- **CI security scans**: `bandit` (SAST for Python), `pip-audit` (CVEs in dependencies),
  Renovate/Dependabot (updates), `trivy`/`grype` (Docker image scan).

**Why it's important:** a leaked secret in git and a vulnerable transitive dependency are the most
frequent security incidents.

---

# Part III. Recommended

## 15. Retry with backoff and circuit breaker

- **Retry** — `tenacity`: exponential backoff + **jitter**, only on
  **idempotent** external calls and only on timeouts/5xx.
- **Circuit breaker** — `pybreaker`/`purgatory`: open the circuit on a series of errors from
  a dependent service, so as not to multiply load and not to fail in a cascade.

**Why:** retries without backoff/jitter amplify an outage (retry storm); without a circuit
breaker, the failure of one dependency drags down the whole service.

---

## 16. Healthcheck: deep vs shallow

Elaboration of §5 of the new file as a separate rule (the crude "healthcheck → 200" from base §3
is insufficient):

- **`/health/live`** — liveness: the process is alive, **without dependencies**.
- **`/health/ready`** — readiness: checks DB/Redis/broker; the pod is taken out of rotation
  when a dependency is unavailable, but **is not restarted**.

---

## 17. DDD tactics (for complex domains)

The architecture is already DDD-light (bounded contexts = `apps/`, UoW). For complex domains —
optional (for simple CRUD this is over-engineering):

- **Value Objects** — Pydantic `frozen=True` models (Email, Money, typed ids)
  instead of primitives.
- **Domain events** — map naturally onto the Outbox (§11).
- **Aggregates** — the boundary of transactional consistency: **UoW = one aggregate per
  transaction**.

---

## 18. Load testing and pool tuning

- **Load tests** — `locust`/`k6` before releasing critical endpoints.
- **Async engine tuning** — explicit `pool_size` / `max_overflow` / `pool_timeout` /
  `pool_pre_ping` for the expected load (core currently has only "the pool as a singleton", §13).

**Why:** the default pool of 5 connections hits its ceiling on the first traffic spike.

---

## 19. DB constraints as data protection

In addition to `index=True`/FK (base §8):

- **`CHECK` / `UNIQUE` / `NOT NULL` at the DB level**, not just Pydantic validation.
- Business invariants expressible in the DB should be in the DB — **the application is not the only
  writer** (migrations, raw SQL, other services).

---

## 20. Background work: workers, consumers, scheduler

> Priority: **Important** — relevant from the first consumer or cron job.

- **The same DI container as HTTP.** Workers, FastStream consumers, and APScheduler
  do not assemble dependencies by hand: for each message/job a
  **REQUEST-scope** dishka container is opened (session, repositories, use cases — as in a request).
- **Cron jobs — under a distributed lock** (Lock from core, base §13): with N
  replicas the job runs once, not N times.
- **A job is idempotent** (re-running is safe) and reduces to **calling a use case**.
  The scheduler/consumer code is the same kind of "controller" as a router: it contains no
  business logic.
- **The transaction is explicit, per unit of work** (message/job), with the same ownership rules
  as in the base rules (§7): the repository does not commit, the boundary is owned by
  the provider/UoW.
- The consumer obeys the event-contract rules ([PLATFORM.md §10](PLATFORM.md)):
  it depends only on the event schema, dedup by `messageId`, DLQ for poison messages.

**Why it's important:** "a scheduler that ran on all replicas" and "a consumer with its own
self-assembled session without rollback" are typical sources of duplicates and half-written data.

---

# Priority summary table

| # | Topic | Priority |
|---|------|-----------|
| 1 | AuthN/AuthZ (where in the architecture, JWT/refresh, Argon2, RBAC/ABAC) | Critical |
| 2 | API security (rate limit, headers, OWASP — BOLA, mass assignment) | Critical |
| 3 | Structured JSON logs + correlation/request id | Critical |
| 4 | Graceful shutdown (SIGTERM) + timeouts on everything | Critical |
| 5 | Prod Dockerfile (multi-stage, non-root) + k8s probes/limits | Critical |
| 6 | Prometheus metrics + OpenTelemetry tracing | Important |
| 7 | Consistent soft delete + created_by/updated_by audit | Important |
| 8 | Optimistic locks (version) + isolation levels | Important |
| 9 | Idempotency of mutations + Idempotency-Key | Important |
| 10 | API: versioning, cursor pagination, filtering standard | Important |
| 11 | Outbox pattern + DLQ + idempotent consumers | Important |
| 12 | Caching: cache-aside, TTL, invalidation, stampede protection | Important |
| 13 | Tests: testcontainers, polyfactory, pyramid, contract | Important |
| 14 | Secrets management (vault) + SAST/SCA (bandit/pip-audit/gitleaks) | Important |
| 15 | Retry+backoff (tenacity) + circuit breaker | Recommended |
| 16 | Healthcheck deep vs shallow (live/ready) | Recommended |
| 17 | DDD tactics: VO, domain events, aggregates | Recommended |
| 18 | Load testing + pool tuning | Recommended |
| 19 | DB constraints (CHECK/UNIQUE/NOT NULL) as data protection | Recommended |
| 20 | Background work: DI scope per job, lock on cron, idempotency | Important |

---

# Quick wins (maximum effect for minimum effort)

1. **`structlog` + `asgi-correlation-id`** (§3) — a couple of hours, radically changes
   the investigability of production incidents.
2. **`extra="forbid"` in Pydantic + resource-ownership check in the UseCase** (§2) —
   closes the two most frequent classes of API vulnerabilities (BOLA, mass assignment) almost for free.
3. **`bandit` + `pip-audit` + `gitleaks`** in CI/pre-commit (§14) — a line each in the config,
   catching secrets and CVEs automatically.

---

# Do / don't summary (production)

**DO**
- AuthZ checks in the domain layer (UseCase/Service); `CurrentUser` via dishka REQUEST-scope.
- Argon2id for passwords; short access + rotatable refresh with revocation.
- Strict Pydantic schemas (`extra="forbid"`); resource-ownership check (BOLA).
- Structured JSON logs + correlation id; logs to stdout.
- Timeouts on all external calls and the DB; graceful shutdown on SIGTERM.
- Multi-stage non-root Dockerfile; separate liveness/readiness probes; resource limits.
- Metrics + tracing with a shared correlation id; Sentry with the release version.
- Soft delete with a partial unique index and a default filter; created_by/updated_by audit.
- Optimistic locking on concurrent updates; a deliberate isolation level.
- Idempotency-Key on critical POSTs; idempotent consumers.
- `/api/v1/` versioning; keyset pagination; a unified filtering contract.
- Outbox on top of UoW; DLQ + retry with backoff for messages.
- Cache-aside with TTL and stampede protection (Lock from core).
- testcontainers + polyfactory; an explicit test pyramid; savepoint rollback.
- Secrets from a vault; bandit/pip-audit/gitleaks/trivy in the pipeline.

**DON'T**
- Check permissions in the router; keep identity in a global contextvars/singleton.
- Store passwords via sha/pbkdf2/md5; eternal JWTs without revocation.
- Accept `id`/`role`/`is_admin` from the request body (mass assignment).
- `print` and unstructured logs; log secrets/PII.
- HTTP/DB calls without a timeout; release without graceful shutdown.
- A root container; a single "200 healthcheck" for both liveness and readiness.
- Offset pagination on large tables; an API without a version.
- Write to the DB and to the broker as two independent operations (dual-write without outbox).
- A cache without TTL and without an invalidation strategy.
- Production secrets in an `.env` file on disk; CI without security scans.
