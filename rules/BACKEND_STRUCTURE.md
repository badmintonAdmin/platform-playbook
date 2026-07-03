# Backend structure (reference)

> The canonical reference structure for a platform backend service. If
> [BACKEND_RULES.md](BACKEND_RULES.md) answers the question **"what are the laws"**, then this
> document answers the question **"how does it look in files and code"**: the concrete tree
> of a domain, the layer skeletons, naming, and — most importantly — **how the dependency graph
> is assembled with dishka and how it is woven into the router**.
>
> This is a template to copy when creating a new domain/service. All entity names
> here are neutral examples (`users`, `User`); substitute the product's problem domain.
> Platform boundaries (API-first, contracts) live in [PLATFORM.md](PLATFORM.md).

---

## 1. Layers and flow direction

A request travels top-down, and the response travels bottom-up. The dependency always points **inward,
toward the abstraction (`Protocol`)**, never toward the implementation.

```
HTTP → Router → UseCase → Service → Repository → [data source]
        (FastAPI) (user story) (domain) (Gateway/I/O)  (Postgres, HTTP, Redis…)

           ← DTO (Pydantic) ←──── layer boundary ────→ DTO (Pydantic) →
```

| Layer | Responsible for | NOT responsible for |
|---|---|---|
| **Router** | declaring the endpoint, calling the UseCase, returning a DTO | business logic, data access |
| **UseCase** | one user story (orchestrating a single operation) | reusable logic, direct I/O |
| **Service** | reusable domain logic, invariants | HTTP, framework awareness |
| **Repository** | any I/O (DB/HTTP/cache), ORM↔DTO mapping | business rules |
| **Model** | ORM entity (only inside the repository) | leaving the repository outward |

Every layer (except models) is described by a pair — **`Protocol` (contract) + `Impl`
(implementation)** — which makes it substitutable under DI and in tests. The whole stack is asynchronous.

**Dependency-direction invariant** (see [BACKEND_RULES.md §4](BACKEND_RULES.md)):
`Controller → UseCase → Service → Repository`, strictly downward. Injecting "upward" is forbidden.

---

## 2. Top level of the service

```
<service>/
├── pyproject.toml          # dependencies and tooling (uv)
├── uv.lock                 # committed
├── alembic.ini
├── .env.example            # configuration template (real values come from the environment)
├── Dockerfile              # multi-stage, non-root (see PRODUCTION §5)
├── docker-compose.yml      # local infrastructure
└── src/
    ├── main.py             # FastAPI app factory, middleware, router mounting
    ├── bootstrap.py        # app assembly: dishka container, exception handlers, lifespan
    ├── settings.py         # pydantic-settings
    ├── cli.py              # Typer commands (seed, maintenance)
    ├── apps/               # DOMAINS (bounded contexts) — reference structure below
    │   ├── health/         #   trimmed-down domain: router + schema
    │   └── users/          #   reference domain
    ├── core/               # platform core: base classes, providers, errors
    ├── migrations/         # Alembic (env.py + versions/)
    └── utils/
```

- **`core/`** — the foundation of all domains (base models, generic repository/service/
  use-case, base exceptions, shared dishka providers). Eventually a separate package.
- **`apps/<domain>/`** — problem domains, each with an identical structure.
- **A new domain becomes visible to the application only after its router and provider are registered**
  (see §7).

---

## 3. Reference structure of a domain

```
apps/users/
├── router.py        # APIRouter(prefix="/users") — the domain exposes ONLY the router outward
├── schemas.py       # Pydantic DTOs: Request/Response + Read/Create/Update
├── use_case.py      # use cases (one user story per operation)
├── services.py      # services (reusable domain logic)
├── repositories.py  # repository protocols + implementations (I/O)
├── models.py        # SQLAlchemy ORM models + enum enumerations
├── exceptions.py    # domain exceptions (inherit the base ones from core)
├── events.py        # (as needed) versioned schemas of domain events — public contract
└── providers.py     # domain dishka providers — bind Protocol → Impl and scope
```

- Every `apps/` domain repeats this structure. The reference is `users`.
- Simple domains (`health`) are trimmed down to `router` + `schema`.
- **Domain isolation** (BACKEND_RULES §4): a domain does not import the internals of another
  domain. A domain's public surface is its `router`, its service/use-case protocols (via DI), and
  `events.py`; everything else is private. Enforced by `import-linter` in CI.

### Granularity: when a file must become a folder

Sprawling god-files (`models.py` with every model, a thousand-line `services.py`) are
the main way to silently kill clean architecture. That's why the split triggers are hard:

- **A layer as a single file — only while the domain has one entity.** As soon as a second
  appears, the layer expands into a folder with one file per entity (`models/order.py`,
  `models/order_item.py`). This is a mechanical `git mv`; postponing it is forbidden.
- **Use cases: one file = one user story** (`use_cases/create_order.py`,
  `use_cases/approve_job.py`). Don't accumulate operations in a shared `use_case.py`.
- **Hard ceiling: file ≤ 400 lines — a CI gate** (exceptions: `migrations/`,
  generated code — via an explicit allowlist). Exceeding it is a signal to split by
  entity/operation, **not to raise the limit**.
- **Complexity via the linter:** Ruff `C901` (cyclomatic), `PLR0912/0913/0915`
  (branches/arguments/statements) are enabled; a long function is refactored, not
  added to the exceptions.
- **Dumping grounds are forbidden:** `utils.py` / `helpers.py` / `misc.py` are not
  created inside a domain. Reusable code goes into `core` as a module with a descriptive name
  (`core/pagination.py`, not `core/utils.py`).
- A large domain may expand its layers into **folders** (`repositories/`, `services/`,
  `use_cases/` with one file per entity) — but the set of layers and the rules stay the same.

| Layer | Required | Form |
|---|---|---|
| `models` | yes | file or folder |
| `schemas` | yes | file or folder |
| `repositories` | yes | file or folder |
| `services` | yes | file or folder |
| `use_case(s)` | yes | file or folder |
| `router` | yes | file or folder |
| `providers` | yes | file |
| `exceptions`, `enums`, `events` | as needed | file |

---

## 4. Layers in code

Example domain `users`. Common vocabulary: `Protocol` — the contract (body `...`), `Impl` — the
implementation.

### `models.py` — ORM entity

```python
from __future__ import annotations
from sqlalchemy.orm import Mapped, mapped_column
from core.models import Base, TimestampMixin

class User(Base, TimestampMixin):          # Base: UUIDv7 PK `id`; TimestampMixin: created_at/updated_at
    __tablename__ = "users"                 # model — singular, table — plural
    email: Mapped[str] = mapped_column(unique=True, index=True)
    is_active: Mapped[bool] = mapped_column(server_default="true")   # booleans — is_ prefix
```

The model **does not leave the repository** — only the DTO goes outward.

### `schemas.py` — DTO (Pydantic v2)

We split by purpose: the use-case boundary (`Request`/`Response`) and the repository boundary
(`Read`/`Create`/`Update`).

```python
class UserReadSchema(BaseSchema):     # from_attributes=True — validation straight from the ORM
    id: UUID
    email: str
    is_active: bool

class UserCreateSchema(BaseSchema):   # id is optional
    email: str

class UserUpdateSchema(BaseSchema):   # id is required, the rest of the fields optional (partial)
    email: str | None = None

class UserResponseSchema(BaseSchema): # outward, camelCase at the boundary (see PLATFORM §3)
    id: UUID
    email: str
```

### `repositories.py` — Gateway (I/O)

```python
class UserRepositoryProtocol(Protocol):
    async def get(self, id: UUID) -> UserReadSchema: ...
    async def create(self, data: UserCreateSchema) -> UserReadSchema: ...

class UserRepositoryImpl(GenericRepository[User], UserRepositoryProtocol):
    def __init__(self, session: AsyncSession) -> None:   # injects ONLY the session
        super().__init__(session, User)
    # generic CRUD is inherited; specific queries are added here
```

**Hard rule:** a DTO goes into the repository and a DTO comes out. ORM→schema mapping is done
via `from_attributes=True`. Relationships are loaded explicitly (`selectinload`, `lazy="raise"`).

### `services.py` — domain logic

```python
class UserServiceProtocol(Protocol):
    async def get_users(self) -> list[UserReadSchema]: ...

class UserServiceImpl(UserServiceProtocol):
    def __init__(self, repo: UserRepositoryProtocol) -> None:   # injects protocols, not Impl
        self._repo = repo

    async def get_users(self) -> list[UserReadSchema]:
        return await self._repo.get_list()
```

### `use_case.py` — user story

A callable class (`async def __call__`). One user story = one operation = one endpoint.
Returns a **response schema**, not an ORM/domain object.

```python
class ListUsersUseCaseProtocol(Protocol):
    async def __call__(self) -> list[UserResponseSchema]: ...

class ListUsersUseCaseImpl(ListUsersUseCaseProtocol):
    def __init__(self, user_service: UserServiceProtocol) -> None:
        self._user_service = user_service

    async def __call__(self) -> list[UserResponseSchema]:
        users = await self._user_service.get_users()
        return [UserResponseSchema.model_validate(u) for u in users]
```

### `router.py` — endpoint

The controller has **no business logic**. Only the UseCase is injected (via dishka).

```python
router = APIRouter(prefix="/users", tags=["users"])

@router.get("")
@inject
async def list_users(
    use_case: FromDishka[ListUsersUseCaseProtocol],
) -> list[UserResponseSchema]:
    return await use_case()
```

---

## 5. Dependency graph with dishka

The DI container is **dishka** (the sole mechanism; `fastapi.Depends` as an IoC container
and hand-rolled factories are forbidden). This is exactly where the dependency graph is built and lives.

### How to read the graph

Real objects depend **downward along the chain**, while the `provides=` binding turns the
dependency arrow toward the **abstraction**:

```
Router  ──FromDishka[ListUsersUseCaseProtocol]──►  UseCaseImpl
                                                       │ needs UserServiceProtocol
                                                       ▼
                                                   ServiceImpl
                                                       │ needs UserRepositoryProtocol
                                                       ▼
                                                   RepositoryImpl
                                                       │ needs AsyncSession
                                                       ▼
                                                   session (generator provider)
                                                       │ needs async_sessionmaker
                                                       ▼
                                                   sessionmaker (APP-scope singleton)
```

dishka resolves this chain itself: to hand the router a `ListUsersUseCaseProtocol`, it
builds the `Impl`, for it the service, for the service the repository, for the repository the session.

### Scope — lifecycle management

| Scope | What lives | Examples |
|---|---|---|
| **`Scope.APP`** | singletons for the whole application | engine, `async_sessionmaker`, configs, pools |
| **`Scope.REQUEST`** | per single request | `AsyncSession`, repositories, services, use cases |

### Domain providers — `providers.py`

Each domain declares its own `Provider`, binding `Impl → Protocol` and setting the scope:

```python
from dishka import Provider, Scope, provide

class UsersProvider(Provider):
    user_repo = provide(
        UserRepositoryImpl, provides=UserRepositoryProtocol, scope=Scope.REQUEST
    )
    user_service = provide(
        UserServiceImpl, provides=UserServiceProtocol, scope=Scope.REQUEST
    )
    list_users_uc = provide(
        ListUsersUseCaseImpl, provides=ListUsersUseCaseProtocol, scope=Scope.REQUEST
    )
```

### Shared providers — `core/`

The session and infrastructure are defined once in the core and reused by all domains:

```python
class CoreProvider(Provider):
    @provide(scope=Scope.APP)                      # singleton: sessionmaker
    def sessionmaker(self, settings: Settings) -> async_sessionmaker[AsyncSession]:
        engine = create_async_engine(settings.db_dsn)
        return async_sessionmaker(engine, expire_on_commit=False)

    @provide(scope=Scope.REQUEST)                  # session + transaction management
    async def session(
        self, maker: async_sessionmaker[AsyncSession]
    ) -> AsyncIterable[AsyncSession]:
        async with maker() as session:
            yield session                          # commit on success, rollback on exception
```

Transaction ownership (the full rule is in BACKEND_RULES §7): **repositories do not commit**
(only `flush`); the default is one transaction per request (this provider); a mutation across
multiple repositories is a Unit of Work, which takes commit/rollback upon itself.

### Assembling the container and weaving it into the router — `bootstrap.py`

```python
from dishka import make_async_container
from dishka.integrations.fastapi import setup_dishka

container = make_async_container(
    CoreProvider(),
    UsersProvider(),                # ← domain registration: add its Provider here
    context={Settings: settings},
)
setup_dishka(container, app)        # binds the container to FastAPI
# await container.close() — in lifespan on shutdown (see §7 and PRODUCTION §4)
```

After `setup_dishka`, any route with `@inject` receives its dependencies via
`FromDishka[...]` (see §4, `router.py`).

### Swapping an implementation — one line

Changing the data source (Postgres → in-memory for tests → HTTP service) is a change of
**a single binding** `provide(..., provides=...)`, with no edits to business logic:

```python
# in the test provider:
user_repo = provide(InMemoryUserRepositoryImpl, provides=UserRepositoryProtocol, scope=Scope.REQUEST)
```

This is precisely why the layers depend on the `Protocol`, not on the `Impl`.

---

## 6. Core `core/`

The foundation from which domains inherit (see also
[BACKEND_RULES.md §13](BACKEND_RULES.md)):

- **Base models:** `Base` (UUIDv7 PK `id`), `BaseInt` (int PK), `TimestampMixin`,
  naming conventions, async engine, session factory.
- **Generic layers:** `GenericRepository` (CRUD for ~90% of cases: `get`/`list`/`create`/
  `update`/`upsert`/`delete`), a base service, a CRUD use-case factory.
- **Base exceptions** + centralized registration of exception handlers (§ below).
- **Shared dishka providers** (session, transaction, healthcheck) and reusable
  services (cryptography, distributed Lock, Unit of Work).
- Connection pools are a **singleton** (APP-scope), otherwise a pool is spun up on every request.

---

## 7. Entry point and lifecycle

### `main.py` — app factory
Creates `FastAPI(title/version from pyproject)`, adds middleware (CORS by allowlist,
correlation-id, security headers — see PRODUCTION §2–3), wires in the domain routers and
`bootstrap.py` (dishka container, exception handlers).

### `bootstrap.py` — assembly
Assembles the dishka container (§5), registers exception handlers (§8), sets up `lifespan`:
on shutdown — graceful (wait for in-flight requests, close pools/brokers, `await
container.close()`, stop the scheduler/consumers; see PRODUCTION §4).

### Registering a new domain — two actions
1. **Router** of the domain — wire it into the aggregator (`app.include_router(users_router)`).
2. **Provider** of the domain — add it to `make_async_container(...)`.

Without both steps, the domain is not visible to the application.

### Workers and scheduler — the same container
Consumers (FastStream) and cron jobs (APScheduler) use **the same
dishka container**: for each message/job a REQUEST scope is opened
(`async with container() as request_container:`), and the use case is pulled from it. Cron runs
under a distributed lock; there is no business logic in the worker code (see PRODUCTION §20).

---

## 8. Exception handling

The hierarchy and rules are in [BACKEND_RULES.md §11](BACKEND_RULES.md). Structurally:

- **Base exceptions live in `core`**, semantic by HTTP meaning (`ModelNotFound →
  404`, `PermissionDenied → 403`, `ModelAlreadyExists → 409`, …). Domain ones live in
  `apps/<domain>/exceptions.py` and inherit the base ones.
- **Before the controller — only business errors** (framework-independent). **After
  the controller — only HTTP errors.** The controller is the boundary where a domain error
  is turned into an HTTP one.
- Mapping to HTTP is done by **centralized FastAPI exception handlers**, one per base
  class (subclasses are covered automatically). Registration happens via a single function from
  `core`.
- **A single error-response envelope** (`error` / `message` / `details`) — a contract for
  all clients (see [PLATFORM.md §3](PLATFORM.md)).

---

## 9. Naming conventions

| Element | Pattern | Example |
|---|---|---|
| Model | `{Entity}` (singular) | `User`, `Order` |
| Model (nested) | `{Parent}{Entity}` | `OrderItem` |
| Table | `{entities}` (plural, snake) | `users`, `order_items` |
| Schema | `{Entity}{Read\|Create\|Update\|Response}Schema` | `UserCreateSchema` |
| Repository | `{Entity}RepositoryProtocol` / `…Impl` | `UserRepositoryProtocol` |
| Service | `{Entity}ServiceProtocol` / `…Impl` | `UserServiceProtocol` |
| UseCase | `{Action}{Entity}UseCaseProtocol` / `…Impl` | `ListUsersUseCaseProtocol` |
| Provider | `{Domain}Provider` | `UsersProvider` |
| Router | `router` variable, prefix `/{entities}` | `/users` |
| File | `snake_case.py` | `order_items.py` |
| PK | always `id` | — |
| Boolean field | `is_` prefix | `is_active` |

An interface uses the `Protocol` suffix; an implementation uses the `Impl` suffix. Domain (expected)
exceptions use the `Error` suffix.

---

## 10. Migrations (Alembic)

The full rules are in [BACKEND_RULES.md §9](BACKEND_RULES.md). The key structural points:

- The **Model First** approach: models → autogenerate DDL.
- In `migrations/env.py`, **all models are imported** and `target_metadata =
  Base.metadata` is assembled. **A new model that isn't imported into `env.py` won't make it into
  autogenerate** — this is the most common slip.
- The migration file name carries a date: `year_month_day_<slug>`.

```bash
uv run alembic upgrade head                      # apply
uv run alembic revision --autogenerate -m "..."  # create
uv run alembic downgrade -1                       # roll back
```

---

## New-domain checklist

1. Create `apps/<domain>/` with the reference structure (§3).
2. Define `models.py` (inherit `Base`/`TimestampMixin`), import it in `env.py`.
3. Define `schemas.py` (Read/Create/Update + Request/Response, `camelCase` outward).
4. `repositories.py`: `Protocol` + `Impl` (inherits `GenericRepository`).
5. `services.py` and `use_case.py`: `Protocol`+`Impl` pairs, injecting **protocols**.
6. `providers.py`: bind all `Impl → Protocol` with a scope (§5).
7. `router.py`: endpoints with `@inject` + `FromDishka[UseCaseProtocol]`.
8. Register the router and `Provider` (§7). Generate the migration.
