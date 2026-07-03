# Rules for building a backend application (Python / FastAPI / Clean Architecture)

> Backend platform ruleset. Describes **how the application should be structured**:
> stack, structure, layers, conventions. Part of the overall ruleset — see
> [README.md](README.md); cross-cutting platform principles (API-first, contract,
> versioning, identity) live in [PLATFORM.md](PLATFORM.md) and **take precedence**.
> The concrete reference structure and dishka graph are in [BACKEND_STRUCTURE.md](BACKEND_STRUCTURE.md).
>
> **Remember: the API serves multiple clients** (web, mobile, integrations) — do not
> design endpoints around a specific frontend (see [PLATFORM.md §1](PLATFORM.md)).
>
> **Architectural decisions adopted:**
> - DI container — **dishka** (replaces hand-written abstract factories and the use of
>   `fastapi.Depends` as an IoC container).
> - Material on **Dev Container** and **debugging inside the container** is not part of these rules.

---

## 1. Stack and tools

| Purpose | Tool | Rule |
|---|---|---|
| Language | **Python 3.13** | Target version. Install the host Python via `pyenv`. |
| Package manager | **uv** (Astral) | The only one. Do not use `pip`/`poetry`. The `uv.lock` lock file is committed and maintained. |
| Web framework | **FastAPI** | Django — for legacy only. |
| DI container | **dishka** | The only dependency injection mechanism (see §10). |
| Validation / DTO | **Pydantic v2** | For schemas/DTOs only. |
| Database access | **SQLAlchemy 2.0** | Used **as a query builder, NOT as an ORM**. `Mapped` / `mapped_column` style. |
| DBMS | **PostgreSQL** | Async driver (`asyncpg`), `AsyncSession`. |
| Migrations | **Alembic** | Model First (Code First) approach. |
| ASGI server | **uvicorn** | Run via `uv run uvicorn`. |
| HTTP client | **HTTPX** | |
| Message brokers | **FastStream** | Kafka / RabbitMQ / NATS / Redis pub-sub. |
| Scheduler | **APScheduler** | Scheduled/interval tasks. |
| ETL | **Prefect** | Replacement for Airflow. |
| Linter | **Ruff** (Astral) | |
| Formatter | **Black** | |
| Typing | **mypy** + **Pyright (strict)** | strict mode is mandatory. |
| Tests | **pytest** | `asyncio_mode = auto`. unittest is not used. |
| Hooks | **pre-commit** | Mandatory (`pre-commit install`). |
| CLI / commands | **Typer** | Single CLI entry point; do not put scripts at the repository root. |
| Containerization | **Docker** + **docker-compose** | Compose — for infrastructure. |

---

## 2. Environment and dependencies (uv)

- All dependencies go through **uv**: `uv sync` builds `.venv` and `uv.lock`.
- `uv.lock` pins the resolved versions; it is **committed to the repository** and not edited by hand.
- Versions in `pyproject.toml` are specified with a caret (`^`).
- Run any project command via `uv run ...`.
- In the IDE, select the interpreter from `.venv` (Select Interpreter).

---

## 3. Project structure

The root package is `src/` (in the boilerplate) or the service name (`todo` → package `todo`).
When forking the boilerplate, `src` is renamed to the project name.

```
<project>/
├── pyproject.toml          # main config: metadata, dependencies, tooling, semantic-release
├── uv.lock                 # committed
├── .python-version
├── .env.example            # copied to .env on clone (cp .env.example .env)
├── .gitignore              # .idea, .venv, __pycache__ — inside; .vscode — NOT inside (committed)
├── .dockerignore           # .env, logging.dev, dev artifacts; migrations DO go into the image
├── alembic.ini
├── docker-compose.yml      # infrastructure (postgres, redis, minio)
├── Dockerfile
├── Makefile                # CLI commands for CI and local
├── README.md
└── src/
    ├── main.py             # entry point: app creation, middleware, main router, Swagger
    ├── bootstrap.py        # app assembly, exception-handler registration, DI container
    ├── settings.py         # pydantic-settings
    ├── cli.py              # Typer commands (seed, encrypt/decrypt, …)
    ├── apps/               # DOMAINS (bounded contexts), each with a uniform structure
    │   ├── health/         # healthcheck: router + schema only (a trimmed-down domain)
    │   └── users/          # reference domain (see below)
    ├── core/               # shared layer for all domains (eventually a separate library)
    ├── migrations/
    │   ├── env.py
    │   └── versions/
    └── utils/
```

### Reference domain structure (`apps/<domain>/`)

```
apps/users/
├── router.py        # APIRouter(prefix="/users") — the module exposes ONLY the router
├── schemas.py       # Pydantic DTOs: request/response + domain schemas
├── use_case.py      # use cases (business logic / user stories)
├── services.py      # services (reusable domain logic)
├── repositories.py  # repository protocols + implementations (I/O)
├── models.py        # SQLAlchemy ORM models + enum types
├── exceptions.py    # domain exceptions (inherit the base ones from core)
└── providers.py     # domain dishka providers (see §10) — replaces the old deps.py
```

- **Every domain in `apps/` has the same structure** (the reference is `users`).
- Simple domains (`health`) are trimmed down to `router` + `schema`.
- **A healthcheck is mandatory** and returns **200** (Readiness / Liveness Probe for k8s/Docker).

### The `tests/` folder

- **Mirrors the structure of `src/`.**
- Test names carry the `test_` prefix: `test_router`, `test_service`, `test_repository`.
- Fixtures live in `conftest.py`. The DB fixture: `alembic upgrade head` before the test and rollback after.

---

## 4. Clean architecture: layers and dependencies

Layers from outermost to innermost:

```
Router (FastAPI)  →  Controller  →  UseCase  →  Service  →  Repository  →  [Data sources]
   (framework)       (entry point)  (user story) (domain)   (Gateway/I/O)   (DB, HTTP, Redis…)
```

**The main rule — Dependency Inversion (the D in SOLID):**
- The dependency points **at the abstraction (Protocol), not at the implementation**.
- The dependency arrow: implementation (`*Impl`) → interface (`*Protocol`).
- The hierarchy is strictly one-directional: `Controller → UseCase → Service → Repository`.

### Rules of the injection chain

1. The controller (route function) contains **no business logic**.
2. **UseCase = one user story** (one user story per endpoint).
3. Only a **UseCase** is injected into the controller.
4. The UseCase injects **services** (and, if needed, repositories).
5. A service injects **other services and repositories**.
6. A repository is self-contained and injects **only the session** (and, questionably, other repositories).
7. **Injecting "upward" is forbidden:** repository → service, service → use case, use case → use case.

### Domain isolation (bounded contexts)

The domains in `apps/` form a modular monolith. To keep it from turning into spaghetti:

- **A domain does not import the internals of another domain** (models, repositories, `Impl`,
  repository-level schemas).
- Domains interact only in two ways:
  1. The **public protocol** of another domain (its Service/UseCase `Protocol`) through the
     DI container;
  2. A **domain event** through a broker/outbox (see [PLATFORM.md §10](PLATFORM.md)).
- **`core` does not import from `apps`.** `apps` imports `core` — never the other way around.
- Isolation is checked **automatically**: `import-linter` in CI (independence contracts
  for domains + layers contracts for layers), not by eyeballing code review.

### Why
The database and the framework are **implementation details** (the outermost layer). Changing the
data source (Postgres → HTTP service → in-memory for tests) or the framework
(FastAPI → Litestar) touches 2–3 files (`providers.py`, `router.py`),
and the business logic is left untouched. Domain isolation additionally preserves the option to split a
domain out into a separate service later — without splitting it out "prematurely" (see PLATFORM §9).

---

## 5. Interfaces, schemas, and DTOs

### Interfaces — via `typing.Protocol`

- Contracts are described via **`typing.Protocol`** (structural typing), **not** ABC.
- Conformance of an implementation to a protocol is checked statically (mypy/Pyright) before runtime.
- Naming: the interface is `<Name>Protocol`, the implementation is `<Name>Impl`.
- The body of a protocol method is `...` (ellipsis), with no implementation.

```python
class UserServiceProtocol(Protocol):
    async def get_users(self) -> list[UserSchema]: ...

class UserServiceImpl:                      # structurally conforms to the protocol
    async def get_users(self) -> list[UserSchema]: ...
```

### DTO — only Pydantic schemas between layers

- **Data is passed between layers ONLY as schemas (Pydantic / dataclass).**
  An ORM object is never raised above the repository.
- Separate schemas by purpose:
  - **Request / Response** — the use case boundary (application layer ↔ the outside world).
  - **Read / Create / Update** — for repositories (see §7).
- Mapping between schemas and objects — `model_validate` / `model_dump`.
- For speed: `model_config = ConfigDict(from_attributes=True)` — validation directly from attributes.
- **Serialization:** the external API uses **camelCase**, inside Python — **snake_case**; schemas convert automatically.

### Sync vs async
- Providers/factories that **just create an object** are synchronous (`def`).
- Working methods (`__call__` on a UseCase, service/repository methods) are **async**.

---

## 6. UseCases and Services

- A **UseCase** is a callable class (`async def __call__`) that reflects a user story and holds business logic.
  It is invoked from the controller: `return await users_use_case()`.
- A **Service** is reusable domain logic that a UseCase calls into.
- **Protocols** of dependencies, not implementations, are injected into a UseCase/Service.
- A UseCase returns a **response schema**, not a domain/ORM object.
- Transactional atomicity across several repositories — the **Unit of Work** pattern
  (`UnitOfWorkProtocol` + `Impl`), which opens a shared transaction.

```python
class UsersUseCaseProtocol(Protocol):
    async def __call__(self) -> list[UserResponseSchema]: ...

class UsersUseCaseImpl:
    def __init__(self, user_service: UserServiceProtocol) -> None:
        self._user_service = user_service

    async def __call__(self) -> list[UserResponseSchema]:
        users = await self._user_service.get_users()
        return [UserResponseSchema.model_validate(u) for u in users]
```

---

## 7. Repositories

- A repository = a **Gateway** that encapsulates **any I/O** (DB, HTTP, Redis, Kafka).
- The abstract repository is a `Protocol` with async methods; implementations are `*Impl`.
- **Hard rule:** a **DTO** goes into the repository and a **DTO** comes out. ORM objects never leave it
  (object-relational impedance mismatch: changing the storage must not break the application).
- Map the ORM model → schema via Pydantic `from_attributes=True`.

### Base generic CRUD repository

A shared parameterized repository covers ~90% of cases. Generic parameters:
the ORM model, the **Read** schema, the **Create** schema, the **Update** schema (the `id` type is inferred from the PK).

- **Read** — `id` is required.
- **Create** — `id` is optional.
- **Update** — `id` is required, the rest of the fields are optional (partial update).

Methods: `get(id)`, `get_list` / `list`, `get_all` (with pagination), `create`, `update`,
`upsert` (via native `INSERT … ON CONFLICT (id) DO UPDATE`), `delete`.
All queries go through `select(...)`.

### Session and transactions — who owns the transaction

There is a single ownership model (without it — partially committed states):

- **A repository never does a `commit`** — only a `flush` (to get an id, to catch a
  constraint). The commit belongs to the boundary, not to the data layer.
- **The default is one transaction per request:** the dishka session provider (§10) commits on a
  successful exit from the request and rolls back on an exception. For simple operations
  (a single aggregate) this is enough — there is no explicit management in business code.
- **A mutation across several repositories — only via Unit of Work** (§6): the UoW explicitly
  opens the transaction and owns commit/rollback; in that case the session provider does not
  commit on its own.
- **Background work** (jobs, consumers) — an explicit transaction per unit of work, the same
  ownership rules (see PRODUCTION §20).
- Obtain the `AsyncSession` **via the dishka provider** (§10), not via a hand-written
  SessionManager. Postgres always runs a query inside a transaction; an "empty" transaction
  collapses.

### Read models for complex reads

The generic CRUD and the UseCase→Service→Repository chain are for domain operations. For
**reports, dashboards, and complex selections**, routing data through all the layers is overkill:

- A **query service** (read-only) is allowed: SQL query → straight to a Response DTO, bypassing the
  domain services and the generic repository. It is a separate class with the same
  `Protocol` + `Impl` pattern, living in the domain next to the repositories.
- **Read-only.** Any mutation goes exclusively through the normal chain via
  Service/Repository/UoW.

### Loading relationships (async!)

- **Lazy loading is forbidden** in async mode. Set **`lazy="raise"`** on relationships —
  it catches accidental lazy accesses.
- Relationships are loaded **explicitly** in the query:
  - **`selectinload`** — recommended (a separate `SELECT … IN (…)`, solves N+1).
  - **`joinedload`** — an alternative (LEFT JOIN).
- Custom method: use `*` as a separator in the signature — required parameters before it,
  optional flags after it (`with_files: bool = False`).

---

## 8. Database models (SQLAlchemy 2.0)

- 2.0 style: `Mapped` / `mapped_column`, `from __future__ import annotations` at the top of the file.
- All models inherit from a common **`Base`** (otherwise they will not land in the registry/metadata).
  - `Base` — a PK of type **UUID**; `BaseInt` — a PK of type **int/bigint**.
- The **`TimestampMixin`** mixin → `created_at` + `updated_at`.

### Naming rules
- **The model is singular** (`File`), **the table is plural** (`files`).
- **The PK is always named `id`.** Properties: uniqueness + minimality.
- **Boolean fields always carry the `is_` prefix** (`is_active`).

### Identifiers (PK)
Order of preference: **UUID v7** → `int` → UUID v4.
- UUID v7 sorts by date like an autoincrement but does not allow counter enumeration (security).
- A sequential int id is vulnerable to enumeration (`order_id` 1,2,3…).

### Columns and defaults
- Always set a **`server_default`** (not only a Python `default`) — for maintaining the DB with raw SQL.
- Dates — type `timestamp`, **everything in UTC**; conversion to the user's timezone happens at display time.
- Add `created_at` / `updated_at` **always** (audit + incremental export to the DWH).

### Enum
- **Do not use native DB ENUM types** — they are painful with migrations (removing a value = a 3-step migration).
- The enum is described on the Python side and stored in the DB as a string/int.

### Typed columns, not a JSON dump
- **An entity is stored as typed columns**, one per field. A table of the form
  `(id, created_at, data jsonb)` with the whole entity in JSON is forbidden: JSONB is not
  indexed for search/filters/sorting/aggregations the way a relational column is.
- **`JSONB` is acceptable only as an ADDITIONAL field** (`extra`/`metadata`) for
  genuinely irregular/heterogeneous data, not as a way to sidestep schema
  design. Fields you search/filter/build reports on are always separate columns
  with indexes.
- Once a field "matures" out of `extra` into something business-important → it is **promoted into a typed
  column** by a migration (expand→contract). `extra` is a waiting room, not a permanent home.

### Relationships
- FK: `ForeignKey("storage.id", ondelete="CASCADE")`.
- **`index=True` on the FK is mandatory** — FKs are not indexed automatically; without an index — a Sequential Scan.
- **`passive_deletes=True`** on the relationship — deletion of children is delegated to the DB (`ON DELETE CASCADE`);
  otherwise SQLAlchemy loads all related objects into memory and fails at large volumes.

---

## 9. Migrations (Alembic)

- **Model First** approach: models → autogenerated DDL.
- In `env.py`, import **all models** (`from ... import *`, so that isort does not break the order),
  and assemble `target_metadata`.
- Migration file name: template `year_month_day_<slug>`.
- Explicitly pass the **DB schema** (`DB_SCHEMA`) — `public` may be disallowed.
- `async_fallback=true` — async with a fallback to sync.
- `use_alter=true` — to break cyclic FK dependencies (the relationship as a separate ALTER).
- When using `sqlalchemy-utils` — add `render_item` (non-standard types).
- Application: `alembic upgrade head`.
- **Expand → contract for breaking DDL.** An incompatible schema change is spread across
  two releases: first **expand** (add the new column/table, migrate the data,
  the code reads both the old and the new), then — in a separate release — **contract** (drop the
  old). Each release's schema is compatible with the previous release's code — deploy and rollback do not
  break running replicas.

---

## 10. DI container — dishka

> Replaces hand-written **abstract factories** and the use of `fastapi.Depends`
> as an IoC container. The dependency inversion principles and Protocol abstractions are preserved;
> only the mechanism of assembly and lifecycle management changes.

### What dishka takes on
1. Creating objects (repositories, services, use cases) by their protocol contract.
2. Binding an implementation to an interface (`provides=Protocol`).
3. **Lifecycle management** via scope (which factories did not close):
   - **`Scope.APP`** — singletons: engine, `async_sessionmaker`, configs, connection pools.
   - **`Scope.REQUEST`** — per request: `AsyncSession`, repositories, services, use cases.
4. Managing the **session and transaction** — via a generator provider (`yield` + commit/rollback).
5. A single point of configuration for the storage provider (Postgres/Redis/in-memory).

### Providers (example)

```python
from dishka import Provider, Scope, provide, make_async_container
from collections.abc import AsyncIterable

class AppProvider(Provider):
    # APP-scope: created once per application
    @provide(scope=Scope.APP)
    def sessionmaker(self, settings: Settings) -> async_sessionmaker[AsyncSession]:
        engine = create_async_engine(settings.db_dsn)
        return async_sessionmaker(engine, expire_on_commit=False)

    # REQUEST-scope: session with transaction management
    @provide(scope=Scope.REQUEST)
    async def session(
        self, maker: async_sessionmaker[AsyncSession]
    ) -> AsyncIterable[AsyncSession]:
        async with maker() as session:
            yield session            # commit/rollback on request exit

    # Binding the implementation to the protocol
    user_repo = provide(
        UserRepositoryImpl, provides=UserRepositoryProtocol, scope=Scope.REQUEST
    )
    user_service = provide(
        UserServiceImpl, provides=UserServiceProtocol, scope=Scope.REQUEST
    )
    users_use_case = provide(
        UsersUseCaseImpl, provides=UsersUseCaseProtocol, scope=Scope.REQUEST
    )
```

### Integration with FastAPI

```python
from dishka import make_async_container
from dishka.integrations.fastapi import setup_dishka, FromDishka, inject

# bootstrap.py
container = make_async_container(AppProvider(), context={Settings: settings})
setup_dishka(container, app)
# closing the container (await container.close()) — in lifespan on shutdown

# router.py — the injection point, only UseCase
@router.get("/users")
@inject
async def get_users(use_case: FromDishka[UsersUseCaseProtocol]) -> list[UserResponseSchema]:
    return await use_case()
```

### Rules
- The controller receives **only a UseCase** via `FromDishka[...]`.
- The choice of implementation (Postgres/Redis/in-memory) — by swapping a single `provide(..., provides=...)` binding,
  with no changes to business logic.
- The container is initialized in `bootstrap.py` and closed in `lifespan`.
- The domain provider file is `providers.py` (instead of the old `deps.py`).

---

## 11. Error handling

### Exception hierarchy
```
Exception (Python)
└── CoreException                        # the root base, in core
    ├── ModelNotFound        → 404
    ├── PermissionDenied     → 403
    ├── ModelAlreadyExists   → 409
    ├── ValidationError      → 422
    └── …                                # one base per "HTTP meaning"
        └── <concrete domain errors>     # in their own domains, inherit the base ones
```

### Rules
- **Base/root exceptions live in `core`.** Domain ones live in `apps/<domain>/exceptions.py`
  and inherit the base ones.
- **Before the controller, only business errors are thrown** (framework-independent).
  **After the controller, only HTTP errors.** The controller is the boundary where a domain error turns into an HTTP one.
- Mapping to HTTP — **centralized FastAPI exception handlers**, one per base class
  (subclasses are covered automatically). Registration — via a single function in `core.exceptions.handlers`.
- **A uniform error response format** (an `error_message` field + an HTTP code) — the frontend knows what to expect.

### Naming
- **Domain (business) exceptions — the `Error` suffix** (expected, "checked").
- **`Exception`** — for system/framework exceptions.
- Base ones — semantic by HTTP meaning (`ModelNotFound`, `PermissionDenied`, …).

---

## 12. Configuration (Settings)

- The settings class inherits from **`BaseSettings`** (pydantic-settings).
- `model_config = SettingsConfigDict(env_file=".env")`.
- Values come from `.env`; the repository holds an **`.env.example`** (`cp .env.example .env`).
  `.env` — in `.gitignore` and `.dockerignore`.
- Boolean fields — explicitly `True`/`False`.
- Typical fields: `DEBUG`, `BASE_URL`, `BASE_DIR`, `SECRET_KEY`, `CORS` (allow_origins),
  `DB_PROVIDER/USER/PASSWORD/HOST/PORT/NAME`, `DB_SCHEMA`. Optional — `SENTRY_DSN`.
- The settings instance is created in `main.py` and passed into the application and into the Alembic `env.py`
  (in the target schema — via the dishka context).

---

## 13. Shared library (core / shared)

- ~80% of the code repeats between services → it is extracted into a shared library ("the core").
- It is included as a **separate package** (PyPI) and versioned with **semantic-release**.
- It contains:
  - **Repositories** (protocol + several implementations): cache (in-memory/Redis),
    DB (in-memory/Postgres), file storage (S3/MinIO — **S3 by default**), a settings repository.
  - **Base models**: `Base` (UUID PK), `BaseInt` (int PK), mixins, naming conventions,
    the async engine, the session factory.
  - **Base exceptions** + registration of handlers.
  - **Services**: cryptography, a distributed Lock (Redis), a transaction service, seed/fixtures.
  - **Schemas**: base request/response, `StatusResponse`.
  - **Utils**: connection pools — **singleton** (otherwise a pool is spun up on every request).
  - **DI providers** (dishka) and a shared healthcheck.
- Create connection pools (DB/thread) as a **singleton**.

---

## 14. Commits and versioning

### Conventional Commits
```
type[optional scope]: description

[body]
[footer]
```
Types: **`feat`** (new functionality), **`fix`** (bugfix), plus `build`, `chore`, `ci`,
`docs`, `style`, `refactor`, `perf`, `test`.

### Breaking changes
- Marked with `!` after the type (`feat!: …`) **or** a `BREAKING CHANGE:` footer.

### SemVer (MAJOR.MINOR.PATCH)
- **MAJOR** — a breaking change (backward compatibility is broken).
- **MINOR** — a `feat` (new functionality without breakage).
- **PATCH** — a `fix` + everything else (`docs`, `chore`, `style`, …).
- A build segment via a hyphen (`10.8.5-1`) — for private Docker registries.

### Automation
- **semantic-release**: on a push to `main` it analyzes the commits, computes the version,
  and releases the image/library. The version-bump commit carries `[skip ci]`.
- → **Meaningful commits following the convention are mandatory** (the version computation depends on them).

---

## 15. Code quality

- **Pyright strict** (`python.analysis.typeCheckingMode = strict`) + **mypy**.
- **Ruff** (lint, autofix) + **Black** (format).
- **isort**: section order future → stdlib → third-party → first-party → local-folder.
- **pre-commit** (`pre-commit install`): hooks before every commit — case/merge-conflict,
  end-of-file/whitespace, black, ruff, type checking. Run all of them: `pre-commit run --all-files`.
- **pytest**: `asyncio_mode = auto`; in pre-commit the tests are usually commented out (they run in CI), coverage.
- Use `assert` freely — under `python -O` they are stripped out and do not slow production down.

---

## 16. Infrastructure (docker-compose)

Services are brought up via docker-compose; the application connects using **service names as hosts**.

- **PostgreSQL** — port `5432`, env `DB_USER/PASSWORD/NAME`, a **healthcheck**.
- **Redis** — port `6379`, cache, a **healthcheck**.
- **MinIO** (S3-compatible, optional) — API `9000`, console `9001`;
  a separate `mc` job creates the bucket. File access in production — **through your own application**, not anonymously.

Rule: **add a healthcheck to every service** (the application waits for its dependencies to be ready).

---

## 17. CI/CD and release

GitHub Actions:
- **On a pull request**: linting (+ tests).
- **On a push to `main`**: linting + tests + release.

Release steps:
1. `checkout`.
2. Install **uv** (action), `uv sync`.
3. **semantic-release** computes the version (config `[tool.semantic_release]` in `pyproject.toml`,
   `version_toml` → the path to the version; commit `[skip ci]`).
4. `upload_to_pypi=false`, `upload_to_release=true` — semantic-release makes a GitHub Release.
5. **build & publish** — as a separate step, only if `outputs.released == 'true'`;
   for PyPI a token is needed (a secret).

---

## 18. Checklist for a new backend service

0. **Design the DB schema** before the code (tables, FKs, `created_at`/`updated_at`,
   soft delete `deleted_at`).
1. Create an **empty** repository on GitHub (no files).
2. Clone the boilerplate, re-point the remote:
   `git remote add origin <url>` → `git branch -M main` → an `init` commit → `git push -u origin main`.
3. Rename `src` to the project name; open it in the editor.
4. `cp .env.example .env`, fill it in (Postgres/Redis/S3/secrets).
5. Bring up docker-compose, check the connections (Postgres, Redis, MinIO + create the buckets `*` and `*-test`).
6. `uv sync`, select the `.venv` interpreter, check the CLI (`python -m cli`).
7. Assemble `main.py`/`bootstrap.py`: metadata from `pyproject.toml` → FastAPI(title/version),
   middleware (CORS, host), **initialization of the dishka container**.
8. Run it (`uv run uvicorn`, port 8000), check `/docs` (Swagger) and `/health`.

---

## Summary of the key "do / don't"

**DO**
- uv, FastAPI, SQLAlchemy 2.0 as a query builder, Pydantic v2, dishka, Alembic.
- Clean architecture: depend on a `Protocol`, not on an implementation; the hierarchy strictly downward.
- Domain isolation: another domain — only through a public protocol or an event;
  `import-linter` in CI.
- Transaction: default — per request (provider); several repositories — UoW; the repository
  does not commit.
- Complex reads — a read-only query service; expand→contract for breaking DDL.
- DTOs between layers; the ORM does not leave the repository.
- PK `id` (UUID v7), the model singular, the table plural, booleans — `is_*`.
- `server_default`, UTC, `created_at`/`updated_at`, `index=True` on FKs, `passive_deletes=True`.
- Async + explicit loading of relationships (`selectinload`, `lazy="raise"`).
- Business errors before the controller, HTTP errors after; centralized handlers; a uniform format.
- Conventional Commits + semantic-release; pre-commit + Pyright strict.
- A healthcheck on every service and endpoint.

**DON'T**
- pip / poetry; Django (except legacy); SQLAlchemy as an ORM.
- Import the internals of another domain; `commit` in a repository; breaking DDL in a single
  release with the code.
- Native ENUM in the DB; lazy loading in async; an int id where security matters.
- ORM objects above the repository; business logic in the controller.
- Hand-written DI factories / `Depends`-as-IoC — only **dishka**.
- Scripts at the project root (only the Typer CLI).
