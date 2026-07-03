# Структура backend (эталонная)

> Каноническая референс-структура backend-сервиса платформы. Если
> [BACKEND_RULES.md](BACKEND_RULES.md) отвечает на вопрос **«какие законы»**, то этот
> документ — на вопрос **«как это выглядит в файлах и коде»**: конкретное дерево
> домена, скелеты слоёв, нейминг, и — главное — **как собирается граф зависимостей на
> dishka и как он вплетается в роутер**.
>
> Это шаблон для копирования при создании нового домена/сервиса. Все имена сущностей
> здесь — нейтральные примеры (`users`, `User`); подставляется предметная область
> продукта. Платформенные границы (API-first, контракт) — в [PLATFORM.md](PLATFORM.md).

---

## 1. Слои и направление потока

Запрос идёт сверху вниз, ответ — снизу вверх. Зависимость всегда направлена **внутрь,
на абстракцию (`Protocol`)**, а не на реализацию.

```
HTTP → Router → UseCase → Service → Repository → [источник данных]
        (FastAPI) (user story) (домен) (Gateway/I/O)  (Postgres, HTTP, Redis…)

           ← DTO (Pydantic) ←──── граница слоёв ────→ DTO (Pydantic) →
```

| Слой | Отвечает за | НЕ отвечает за |
|---|---|---|
| **Router** | объявить endpoint, вызвать UseCase, вернуть DTO | бизнес-логику, доступ к данным |
| **UseCase** | одну user story (оркестрация одной операции) | переиспользуемую логику, I/O напрямую |
| **Service** | переиспользуемую доменную логику, инварианты | HTTP, знание про фреймворк |
| **Repository** | любой I/O (БД/HTTP/кэш), маппинг ORM↔DTO | бизнес-правила |
| **Model** | ORM-сущность (только внутри репозитория) | покидать репозиторий наружу |

Каждый слой (кроме моделей) описан парой **`Protocol` (контракт) + `Impl`
(реализация)** — это даёт подменяемость при DI и в тестах. Весь стек асинхронный.

**Инвариант направления зависимостей** (см. [BACKEND_RULES.md §4](BACKEND_RULES.md)):
`Controller → UseCase → Service → Repository`, строго вниз. Инжектить «вверх» запрещено.

---

## 2. Верхний уровень сервиса

```
<service>/
├── pyproject.toml          # зависимости и тулинг (uv)
├── uv.lock                 # коммитится
├── alembic.ini
├── .env.example            # шаблон конфигурации (реальные значения — из окружения)
├── Dockerfile              # multi-stage, non-root (см. PRODUCTION §5)
├── docker-compose.yml      # локальная инфраструктура
└── src/
    ├── main.py             # фабрика FastAPI-приложения, middleware, монтаж роутеров
    ├── bootstrap.py        # сборка приложения: dishka-контейнер, exception handlers, lifespan
    ├── settings.py         # pydantic-settings
    ├── cli.py              # Typer-команды (seed, обслуживание)
    ├── apps/               # ДОМЕНЫ (bounded contexts) — эталонная структура ниже
    │   ├── health/         #   урезанный домен: router + schema
    │   └── users/          #   эталонный домен
    ├── core/               # платформенное ядро: базовые классы, провайдеры, ошибки
    ├── migrations/         # Alembic (env.py + versions/)
    └── utils/
```

- **`core/`** — фундамент всех доменов (базовые модели, generic-репозиторий/сервис/
  use-case, базовые исключения, общие dishka-провайдеры). В перспективе — отдельный пакет.
- **`apps/<domain>/`** — предметные области, каждая с одинаковой структурой.
- **Новый домен виден приложению только после регистрации его роутера и провайдера**
  (см. §7).

---

## 3. Эталонная структура домена

```
apps/users/
├── router.py        # APIRouter(prefix="/users") — наружу из домена отдаётся ТОЛЬКО router
├── schemas.py       # Pydantic DTO: Request/Response + Read/Create/Update
├── use_case.py      # use cases (по одной user story на operation)
├── services.py      # сервисы (переиспользуемая доменная логика)
├── repositories.py  # протоколы репозиториев + реализации (I/O)
├── models.py        # ORM-модели SQLAlchemy + enum-перечисления
├── exceptions.py    # доменные исключения (наследуют базовые из core)
├── events.py        # (по необходимости) версионируемые схемы доменных событий — публичный контракт
└── providers.py     # dishka-провайдеры домена — связывают Protocol → Impl и scope
```

- Каждый домен `apps/` повторяет эту структуру. Эталон — `users`.
- Простые домены (`health`) урезаются до `router` + `schema`.
- **Изоляция доменов** (BACKEND_RULES §4): домен не импортирует внутренности другого
  домена. Публичное у домена — `router`, протоколы сервисов/use case'ов (через DI) и
  `events.py`; всё остальное — приватное. Проверяется `import-linter` в CI.

### Гранулярность: когда файл обязан стать папкой

Разросшиеся god-файлы (`models.py` со всеми моделями, `services.py` на тысячу строк) —
главный способ убить чистую архитектуру молча. Поэтому триггеры разделения — жёсткие:

- **Слой одним файлом — только пока в домене одна сущность.** Появилась вторая —
  слой разворачивается в папку с файлом на сущность (`models/order.py`,
  `models/order_item.py`). Это механический `git mv`, откладывать запрещено.
- **Use cases: один файл = одна user story** (`use_cases/create_order.py`,
  `use_cases/approve_job.py`). Не копить операции в общем `use_case.py`.
- **Жёсткий потолок: файл ≤ 400 строк — CI-гейт** (исключения: `migrations/`,
  сгенерированный код — явным allowlist'ом). Превышение = сигнал делить по
  сущности/операции, а **не поднимать лимит**.
- **Сложность — линтером:** Ruff `C901` (цикломатическая), `PLR0912/0913/0915`
  (ветвления/аргументы/стейтменты) включены; длинная функция рефакторится, а не
  добавляется в исключения.
- **Свалки запрещены:** `utils.py` / `helpers.py` / `misc.py` внутри домена не
  заводятся. Переиспользуемое — в `core` модулем с говорящим именем
  (`core/pagination.py`, а не `core/utils.py`).
- Крупный домен может разворачивать слои в **папки** (`repositories/`, `services/`,
  `use_cases/` с файлом на сущность) — но состав слоёв и правила те же.

| Слой | Обязателен | Оформление |
|---|---|---|
| `models` | да | файл или папка |
| `schemas` | да | файл или папка |
| `repositories` | да | файл или папка |
| `services` | да | файл или папка |
| `use_case(s)` | да | файл или папка |
| `router` | да | файл или папка |
| `providers` | да | файл |
| `exceptions`, `enums`, `events` | по необходимости | файл |

---

## 4. Слои в коде

Пример домена `users`. Общий словарь: `Protocol` — контракт (тело `...`), `Impl` —
реализация.

### `models.py` — ORM-сущность

```python
from __future__ import annotations
from sqlalchemy.orm import Mapped, mapped_column
from core.models import Base, TimestampMixin

class User(Base, TimestampMixin):          # Base: UUIDv7 PK `id`; TimestampMixin: created_at/updated_at
    __tablename__ = "users"                 # модель — ед. число, таблица — мн. число
    email: Mapped[str] = mapped_column(unique=True, index=True)
    is_active: Mapped[bool] = mapped_column(server_default="true")   # булевы — префикс is_
```

Модель **не покидает репозиторий** — наружу идёт только DTO.

### `schemas.py` — DTO (Pydantic v2)

Разделяем по назначению: граница use case (`Request`/`Response`) и граница репозитория
(`Read`/`Create`/`Update`).

```python
class UserReadSchema(BaseSchema):     # from_attributes=True — валидация прямо из ORM
    id: UUID
    email: str
    is_active: bool

class UserCreateSchema(BaseSchema):   # id опционален
    email: str

class UserUpdateSchema(BaseSchema):   # id обязателен, остальные поля опциональны (partial)
    email: str | None = None

class UserResponseSchema(BaseSchema): # наружу, camelCase на границе (см. PLATFORM §3)
    id: UUID
    email: str
```

### `repositories.py` — Gateway (I/O)

```python
class UserRepositoryProtocol(Protocol):
    async def get(self, id: UUID) -> UserReadSchema: ...
    async def create(self, data: UserCreateSchema) -> UserReadSchema: ...

class UserRepositoryImpl(GenericRepository[User], UserRepositoryProtocol):
    def __init__(self, session: AsyncSession) -> None:   # инжектит ТОЛЬКО сессию
        super().__init__(session, User)
    # generic CRUD наследуется; специфичные запросы дописываются здесь
```

**Жёсткое правило:** в репозиторий входит DTO и выходит DTO. Маппинг ORM→схема —
через `from_attributes=True`. Связи грузятся явно (`selectinload`, `lazy="raise"`).

### `services.py` — доменная логика

```python
class UserServiceProtocol(Protocol):
    async def get_users(self) -> list[UserReadSchema]: ...

class UserServiceImpl(UserServiceProtocol):
    def __init__(self, repo: UserRepositoryProtocol) -> None:   # инжектит протоколы, не Impl
        self._repo = repo

    async def get_users(self) -> list[UserReadSchema]:
        return await self._repo.get_list()
```

### `use_case.py` — user story

Callable-класс (`async def __call__`). Одна user story = одна операция = один endpoint.
Возвращает **response-схему**, не ORM/доменный объект.

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

В контроллере **нет бизнес-логики**. Инжектится **только UseCase** (через dishka).

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

## 5. Граф зависимостей на dishka

DI-контейнер — **dishka** (единственный механизм; `fastapi.Depends` как IoC-контейнер
и самописные фабрики — запрещены). Именно здесь строится и живёт граф зависимостей.

### Как читать граф

Реальные объекты зависят **вниз по цепочке**, а привязка `provides=` разворачивает
стрелку зависимости на **абстракцию**:

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
                                                   session (провайдер-генератор)
                                                       │ needs async_sessionmaker
                                                       ▼
                                                   sessionmaker (APP-scope singleton)
```

dishka сам разрешает эту цепочку: чтобы отдать роутеру `ListUsersUseCaseProtocol`, он
строит `Impl`, для него — сервис, для сервиса — репозиторий, для репозитория — сессию.

### Scope — управление жизненным циклом

| Scope | Что живёт | Примеры |
|---|---|---|
| **`Scope.APP`** | синглтоны на всё приложение | engine, `async_sessionmaker`, конфиги, пулы |
| **`Scope.REQUEST`** | на один запрос | `AsyncSession`, репозитории, сервисы, use cases |

### Провайдеры домена — `providers.py`

Каждый домен объявляет свой `Provider`, связывающий `Impl → Protocol` и задающий scope:

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

### Общие провайдеры — `core/`

Сессия и инфраструктура — один раз в ядре, переиспользуются всеми доменами:

```python
class CoreProvider(Provider):
    @provide(scope=Scope.APP)                      # singleton: sessionmaker
    def sessionmaker(self, settings: Settings) -> async_sessionmaker[AsyncSession]:
        engine = create_async_engine(settings.db_dsn)
        return async_sessionmaker(engine, expire_on_commit=False)

    @provide(scope=Scope.REQUEST)                  # сессия + управление транзакцией
    async def session(
        self, maker: async_sessionmaker[AsyncSession]
    ) -> AsyncIterable[AsyncSession]:
        async with maker() as session:
            yield session                          # commit при успехе, rollback при исключении
```

Владение транзакцией (полное правило — BACKEND_RULES §7): **репозитории не коммитят**
(только `flush`); дефолт — одна транзакция на запрос (этот провайдер); мутация через
несколько репозиториев — Unit of Work, который берёт commit/rollback на себя.

### Сборка контейнера и вплетение в роутер — `bootstrap.py`

```python
from dishka import make_async_container
from dishka.integrations.fastapi import setup_dishka

container = make_async_container(
    CoreProvider(),
    UsersProvider(),                # ← регистрация домена: добавить его Provider сюда
    context={Settings: settings},
)
setup_dishka(container, app)        # связывает контейнер с FastAPI
# await container.close() — в lifespan при остановке (см. §7 и PRODUCTION §4)
```

После `setup_dishka` любой роут с `@inject` получает зависимости через
`FromDishka[...]` (см. §4, `router.py`).

### Подмена реализации — одна строка

Смена источника данных (Postgres → in-memory для тестов → HTTP-сервис) — это смена
**одной привязки** `provide(..., provides=...)`, без правок бизнес-логики:

```python
# в тестовом провайдере:
user_repo = provide(InMemoryUserRepositoryImpl, provides=UserRepositoryProtocol, scope=Scope.REQUEST)
```

Именно ради этого слои зависят от `Protocol`, а не от `Impl`.

---

## 6. Ядро `core/`

Фундамент, от которого наследуются домены (см. также
[BACKEND_RULES.md §13](BACKEND_RULES.md)):

- **Базовые модели:** `Base` (UUIDv7 PK `id`), `BaseInt` (int PK), `TimestampMixin`,
  naming conventions, async engine, фабрика сессий.
- **Generic-слои:** `GenericRepository` (CRUD ~90% случаев: `get`/`list`/`create`/
  `update`/`upsert`/`delete`), базовый сервис, фабрика CRUD-use-case'ов.
- **Базовые исключения** + централизованная регистрация exception handlers (§ниже).
- **Общие dishka-провайдеры** (сессия, транзакция, healthcheck) и переиспользуемые
  сервисы (криптография, распределённый Lock, Unit of Work).
- Пулы соединений — **singleton** (APP-scope), иначе пул поднимается на каждый запрос.

---

## 7. Точка входа и жизненный цикл

### `main.py` — фабрика приложения
Создаёт `FastAPI(title/version из pyproject)`, добавляет middleware (CORS по allowlist,
correlation-id, security headers — см. PRODUCTION §2–3), подключает роутеры доменов и
`bootstrap.py` (dishka-контейнер, exception handlers).

### `bootstrap.py` — сборка
Собирает dishka-контейнер (§5), регистрирует exception handlers (§8), задаёт `lifespan`:
на shutdown — graceful (дождаться in-flight, закрыть пулы/брокеры, `await
container.close()`, остановить планировщик/консьюмеров; см. PRODUCTION §4).

### Регистрация нового домена — два действия
1. **Роутер** домена подключить в агрегатор (`app.include_router(users_router)`).
2. **Provider** домена добавить в `make_async_container(...)`.

Без обоих шагов домен приложению не виден.

### Воркеры и планировщик — тот же контейнер
Консьюмеры (FastStream) и cron-задачи (APScheduler) используют **тот же
dishka-контейнер**: на каждое сообщение/джобу открывается REQUEST-scope
(`async with container() as request_container:`), из него достаётся use case. Cron —
под распределённым локом; бизнес-логики в коде воркера нет (см. PRODUCTION §20).

---

## 8. Обработка исключений

Иерархия и правила — в [BACKEND_RULES.md §11](BACKEND_RULES.md). Структурно:

- **Базовые исключения — в `core`**, семантические по HTTP-смыслу (`ModelNotFound →
  404`, `PermissionDenied → 403`, `ModelAlreadyExists → 409`, …). Доменные — в
  `apps/<domain>/exceptions.py`, наследуют базовые.
- **До контроллера — только бизнес-ошибки** (фреймворк-независимые). **После
  контроллера — только HTTP-ошибки.** Контроллер — граница превращения доменной ошибки
  в HTTP.
- Маппинг в HTTP — **централизованные FastAPI exception handlers**, по одному на базовый
  класс (наследники покрываются автоматически). Регистрация — единой функцией из
  `core`.
- **Единый конверт ответа об ошибке** (`error` / `message` / `details`) — контракт для
  всех клиентов (см. [PLATFORM.md §3](PLATFORM.md)).

---

## 9. Конвенции именования

| Элемент | Шаблон | Пример |
|---|---|---|
| Model | `{Entity}` (ед. число) | `User`, `Order` |
| Model (вложенная) | `{Parent}{Entity}` | `OrderItem` |
| Таблица | `{entities}` (мн. число, snake) | `users`, `order_items` |
| Schema | `{Entity}{Read\|Create\|Update\|Response}Schema` | `UserCreateSchema` |
| Repository | `{Entity}RepositoryProtocol` / `…Impl` | `UserRepositoryProtocol` |
| Service | `{Entity}ServiceProtocol` / `…Impl` | `UserServiceProtocol` |
| UseCase | `{Action}{Entity}UseCaseProtocol` / `…Impl` | `ListUsersUseCaseProtocol` |
| Provider | `{Domain}Provider` | `UsersProvider` |
| Router | переменная `router`, префикс `/{entities}` | `/users` |
| Файл | `snake_case.py` | `order_items.py` |
| PK | всегда `id` | — |
| Булево поле | префикс `is_` | `is_active` |

Интерфейс — суффикс `Protocol`; реализация — суффикс `Impl`. Доменные (ожидаемые)
исключения — суффикс `Error`.

---

## 10. Миграции (Alembic)

Полные правила — [BACKEND_RULES.md §9](BACKEND_RULES.md). Ключевое структурно:

- Подход **Model First**: модели → autogenerate DDL.
- В `migrations/env.py` **импортируются все модели** и собирается `target_metadata =
  Base.metadata`. **Новая модель, не импортированная в `env.py`, не попадёт в
  autogenerate** — это самый частый промах.
- Имя файла миграции с датой: `год_месяц_день_<slug>`.

```bash
uv run alembic upgrade head                      # применить
uv run alembic revision --autogenerate -m "..."  # создать
uv run alembic downgrade -1                       # откатить
```

---

## Чеклист нового домена

1. Создать `apps/<domain>/` с эталонной структурой (§3).
2. Описать `models.py` (наследовать `Base`/`TimestampMixin`), импортировать в `env.py`.
3. Описать `schemas.py` (Read/Create/Update + Request/Response, `camelCase` наружу).
4. `repositories.py`: `Protocol` + `Impl` (наследует `GenericRepository`).
5. `services.py` и `use_case.py`: пары `Protocol`+`Impl`, инжектят **протоколы**.
6. `providers.py`: связать все `Impl → Protocol` со scope (§5).
7. `router.py`: endpoints с `@inject` + `FromDishka[UseCaseProtocol]`.
8. Зарегистрировать роутер и `Provider` (§7). Сгенерировать миграцию.
