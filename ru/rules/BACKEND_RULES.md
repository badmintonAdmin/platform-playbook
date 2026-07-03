# Правила создания backend-приложения (Python / FastAPI / Чистая архитектура)

> Свод правил backend платформы. Описывает, **как должно быть устроено** приложение:
> стек, структура, слои, соглашения. Часть общего свода — см.
> [README.md](README.md); сквозные платформенные принципы (API-first, контракт,
> версионирование, идентичность) — в [PLATFORM.md](PLATFORM.md) и **имеют приоритет**.
> Конкретная референс-структура и dishka-граф — в [BACKEND_STRUCTURE.md](BACKEND_STRUCTURE.md).
>
> **Помни: API обслуживает несколько клиентов** (web, mobile, интеграции) — не
> проектируй эндпоинты под конкретный фронт (см. [PLATFORM.md §1](PLATFORM.md)).
>
> **Принятые архитектурные решения:**
> - DI-контейнер — **dishka** (заменяет самописные абстрактные фабрики и
>   использование `fastapi.Depends` как IoC-контейнера).
> - Материал про **Dev Container** и **отладку в контейнере** в правила не входит.

---

## 1. Стек и инструменты

| Назначение | Инструмент | Правило |
|---|---|---|
| Язык | **Python 3.13** | Целевая версия. Установка хостового Python — через `pyenv`. |
| Менеджер пакетов | **uv** (Astral) | Единственный. `pip`/`poetry` не использовать. Lock-файл `uv.lock` коммитится и бережётся. |
| Web-фреймворк | **FastAPI** | Django — только для legacy. |
| DI-контейнер | **dishka** | Единственный механизм внедрения зависимостей (см. §10). |
| Валидация / DTO | **Pydantic v2** | Только для схем/DTO. |
| Работа с БД | **SQLAlchemy 2.0** | Используется **как query builder, НЕ как ORM**. Стиль `Mapped` / `mapped_column`. |
| СУБД | **PostgreSQL** | Async-драйвер (`asyncpg`), `AsyncSession`. |
| Миграции | **Alembic** | Подход Model First (Code First). |
| ASGI-сервер | **uvicorn** | Запуск через `uv run uvicorn`. |
| HTTP-клиент | **HTTPX** | |
| Брокеры сообщений | **FastStream** | Kafka / RabbitMQ / NATS / Redis pub-sub. |
| Планировщик | **APScheduler** | Задачи по расписанию/интервалу. |
| ETL | **Prefect** | Замена Airflow. |
| Линтер | **Ruff** (Astral) | |
| Форматтер | **Black** | |
| Типизация | **mypy** + **Pyright (strict)** | strict-режим обязателен. |
| Тесты | **pytest** | `asyncio_mode = auto`. unittest не используется. |
| Хуки | **pre-commit** | Обязательно (`pre-commit install`). |
| CLI / команды | **Typer** | Единая точка входа CLI; скрипты в корне не класть. |
| Контейнеризация | **Docker** + **docker-compose** | Compose — для инфраструктуры. |

---

## 2. Окружение и зависимости (uv)

- Все зависимости через **uv**: `uv sync` собирает `.venv` и `uv.lock`.
- `uv.lock` фиксирует разрешённые версии; **коммитится в репозиторий**, вручную не правится.
- Версии в `pyproject.toml` задаются с caret (`^`).
- Запуск любых команд проекта — через `uv run ...`.
- В IDE выбирать интерпретатор из `.venv` (Select Interpreter).

---

## 3. Структура проекта

Корневой пакет — `src/` (в boilerplate) или имя сервиса (`todo` → пакет `todo`).
При форке boilerplate `src` переименовывается в имя проекта.

```
<project>/
├── pyproject.toml          # главный конфиг: метаданные, зависимости, тулинг, semantic-release
├── uv.lock                 # коммитится
├── .python-version
├── .env.example            # копируется в .env при клонировании (cp .env.example .env)
├── .gitignore              # .idea, .venv, __pycache__ — внутри; .vscode — НЕ внутри (коммитится)
├── .dockerignore           # .env, logging.dev, dev-артефакты; миграции В образ попадают
├── alembic.ini
├── docker-compose.yml      # инфраструктура (postgres, redis, minio)
├── Dockerfile
├── Makefile                # CLI-команды для CI и локали
├── README.md
└── src/
    ├── main.py             # точка входа: создание app, middleware, главный роутер, Swagger
    ├── bootstrap.py        # сборка приложения, регистрация exception handlers, DI-контейнер
    ├── settings.py         # pydantic-settings
    ├── cli.py              # Typer-команды (seed, encrypt/decrypt, …)
    ├── apps/               # ДОМЕНЫ (bounded contexts), у каждого единая структура
    │   ├── health/         # healthcheck: только router + schema (урезанный домен)
    │   └── users/          # эталонный домен (см. ниже)
    ├── core/               # общий слой для всех доменов (в перспективе — отдельная библиотека)
    ├── migrations/
    │   ├── env.py
    │   └── versions/
    └── utils/
```

### Эталонная структура домена (`apps/<domain>/`)

```
apps/users/
├── router.py        # APIRouter(prefix="/users") — наружу из модуля отдаётся ТОЛЬКО router
├── schemas.py       # Pydantic DTO: request/response + доменные схемы
├── use_case.py      # use cases (бизнес-логика / user stories)
├── services.py      # сервисы (переиспользуемая доменная логика)
├── repositories.py  # протоколы репозиториев + реализации (I/O)
├── models.py        # ORM-модели SQLAlchemy + enum-перечисления
├── exceptions.py    # доменные исключения (наследуют базовые из core)
└── providers.py     # dishka-провайдеры домена (см. §10) — заменяет старый deps.py
```

- **Каждый домен в `apps/` имеет одинаковую структуру** (эталон — `users`).
- Простые домены (`health`) урезаются до `router` + `schema`.
- **Healthcheck обязателен**, возвращает **200** (Readiness / Liveness Probe для k8s/Docker).

### Папка `tests/`

- **Зеркалит структуру `src/`.**
- Имена тестов с префиксом `test_`: `test_router`, `test_service`, `test_repository`.
- Фикстуры — в `conftest.py`. Фикстура БД: `alembic upgrade head` перед тестом и откат после.

---

## 4. Чистая архитектура: слои и зависимости

Слои от внешнего к внутреннему:

```
Router (FastAPI)  →  Controller  →  UseCase  →  Service  →  Repository  →  [Источники данных]
   (фреймворк)       (точка входа)  (user story) (доменная)  (Gateway/I/O)   (БД, HTTP, Redis…)
```

**Главное правило — Dependency Inversion (буква D из SOLID):**
- Зависимость направлена **на абстракцию (Protocol), а не на реализацию**.
- Стрелка зависимости: реализация (`*Impl`) → интерфейс (`*Protocol`).
- Иерархия строго однонаправленная: `Controller → UseCase → Service → Repository`.

### Правила цепочки внедрения

1. В контроллере (роут-функции) **нет бизнес-логики**.
2. **UseCase = одна user story** (на один endpoint — одна user story).
3. В контроллер инжектится **только UseCase**.
4. UseCase инжектит **сервисы** (и при необходимости репозитории).
5. Сервис инжектит **другие сервисы и репозитории**.
6. Репозиторий самостоятелен, инжектит **только сессию** (и, под вопросом, другие репозитории).
7. **Запрещено инжектить «вверх»:** репозиторий → сервис, сервис → use case, use case → use case.

### Изоляция доменов (bounded contexts)

Домены в `apps/` — модульный монолит. Чтобы он не превратился в спагетти:

- **Домен не импортирует внутренности другого домена** (модели, репозитории, `Impl`,
  схемы репозиторного уровня).
- Взаимодействие доменов — только двумя способами:
  1. **Публичный протокол** другого домена (его Service/UseCase `Protocol`) через
     DI-контейнер;
  2. **Доменное событие** через брокер/outbox (см. [PLATFORM.md §10](PLATFORM.md)).
- **`core` не импортирует из `apps`.** `apps` импортирует `core` — никогда наоборот.
- Изоляция проверяется **автоматически**: `import-linter` в CI (контракты
  independence для доменов + layers для слоёв), а не код-ревью «на глаз».

### Зачем
БД и фреймворк — **детали реализации** (самый внешний слой). Смена источника данных
(Postgres → HTTP-сервис → in-memory для тестов) или фреймворка
(FastAPI → Litestar) меняет 2–3 файла (`providers.py`, `router.py`),
а бизнес-логика не трогается. Изоляция доменов дополнительно сохраняет право выделить
домен в отдельный сервис позже — не выделяя его «на вырост» (см. PLATFORM §9).

---

## 5. Интерфейсы, схемы и DTO

### Интерфейсы — через `typing.Protocol`

- Контракты описываются через **`typing.Protocol`** (структурная типизация), **не** ABC.
- Соответствие реализации протоколу проверяется статически (mypy/Pyright) до рантайма.
- Именование: интерфейс — `<Name>Protocol`, реализация — `<Name>Impl`.
- Тело методов протокола — `...` (эллипсис), без реализации.

```python
class UserServiceProtocol(Protocol):
    async def get_users(self) -> list[UserSchema]: ...

class UserServiceImpl:                      # подчиняется протоколу структурно
    async def get_users(self) -> list[UserSchema]: ...
```

### DTO — только Pydantic-схемы между слоями

- **Между слоями данные передаются ТОЛЬКО схемами (Pydantic / dataclass).**
  ORM-объект никогда не поднимается выше репозитория.
- Разделяй схемы по назначению:
  - **Request / Response** — граница use case (слой application ↔ внешний мир).
  - **Read / Create / Update** — для репозиториев (см. §7).
- Маппинг между схемами и объектами — `model_validate` / `model_dump`.
- Для скорости: `model_config = ConfigDict(from_attributes=True)` — валидация прямо из атрибутов.
- **Сериализация:** внешний API — **camelCase**, внутри Python — **snake_case**; схемы конвертируют автоматически.

### Sync vs async
- Провайдеры/фабрики, которые **просто создают объект**, — синхронные (`def`).
- Рабочие методы (`__call__` у UseCase, методы сервисов/репозиториев) — **async**.

---

## 6. UseCases и Services

- **UseCase** — callable-класс (`async def __call__`), отражает user story, содержит бизнес-логику.
  Вызывается из контроллера: `return await users_use_case()`.
- **Service** — переиспользуемая доменная логика, к которой обращается UseCase.
- В UseCase/Service инжектятся **протоколы** зависимостей, не реализации.
- UseCase возвращает **response-схему**, не доменный/ORM-объект.
- Транзакционная атомарность нескольких репозиториев — паттерн **Unit of Work**
  (`UnitOfWorkProtocol` + `Impl`), открывающий общую транзакцию.

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

## 7. Репозитории

- Репозиторий = **Gateway**, инкапсулирует **любой I/O** (БД, HTTP, Redis, Kafka).
- Абстрактный репозиторий — `Protocol` с async-методами; реализации — `*Impl`.
- **Жёсткое правило:** в репозиторий входит **DTO** и выходит **DTO**. ORM-объекты наружу не выходят
  (объектно-реляционный разрыв: смена хранилища не должна ломать приложение).
- Маппинг ORM-модель → схема через Pydantic `from_attributes=True`.

### Базовый generic CRUD-репозиторий

Общий параметризуемый репозиторий покрывает ~90% случаев. Generic-параметры:
ORM-модель, **Read**-схема, **Create**-схема, **Update**-схема (тип `id` выводится из PK).

- **Read** — `id` обязателен.
- **Create** — `id` опционален.
- **Update** — `id` обязателен, остальные поля опциональны (частичный апдейт).

Методы: `get(id)`, `get_list` / `list`, `get_all` (с пагинацией), `create`, `update`,
`upsert` (через нативный `INSERT … ON CONFLICT (id) DO UPDATE`), `delete`.
Все запросы — через `select(...)`.

### Сессия и транзакции — кто владеет транзакцией

Единственная модель владения (без неё — полузакоммиченные состояния):

- **Репозиторий никогда не делает `commit`** — только `flush` (получить id, поймать
  constraint). Commit принадлежит границе, а не слою данных.
- **Дефолт — одна транзакция на запрос:** dishka-провайдер сессии (§10) коммитит при
  успешном выходе из запроса и откатывает при исключении. Для простых операций
  (один агрегат) этого достаточно — явного управления в бизнес-коде нет.
- **Мутация через несколько репозиториев — только через Unit of Work** (§6): UoW явно
  открывает транзакцию, владеет commit/rollback; провайдер сессии в этом случае не
  коммитит сам.
- **Фоновая работа** (джобы, консьюмеры) — явная транзакция на единицу работы, те же
  правила владения (см. PRODUCTION §20).
- Получение `AsyncSession` — **через dishka-провайдер** (§10), не через самописный
  SessionManager. Postgres всегда выполняет запрос в транзакции; «пустая» транзакция
  схлопывается.

### Read-модели для сложных чтений

Generic CRUD и цепочка UseCase→Service→Repository — для доменных операций. Для
**отчётов, дашбордов, сложных выборок** гонять данные через все слои — лишнее:

- Разрешён **query-сервис** (read-only): SQL-запрос → сразу Response-DTO, минуя
  доменные сервисы и generic-репозиторий. Это отдельный класс с тем же паттерном
  `Protocol` + `Impl`, живёт в домене рядом с репозиториями.
- **Только чтение.** Любая мутация — исключительно обычной цепочкой через
  Service/Repository/UoW.

### Загрузка связей (async!)

- **Lazy loading запрещён** в async-режиме. На relationship ставить **`lazy="raise"`** —
  ловит случайные ленивые обращения.
- Связи грузятся **явно** в запросе:
  - **`selectinload`** — рекомендуемый (отдельный `SELECT … IN (…)`, решает N+1).
  - **`joinedload`** — альтернатива (LEFT JOIN).
- Кастомный метод: разделитель `*` в сигнатуре — до него обязательные параметры,
  после — опциональные флаги (`with_files: bool = False`).

---

## 8. Модели БД (SQLAlchemy 2.0)

- Стиль 2.0: `Mapped` / `mapped_column`, `from __future__ import annotations` в начале файла.
- Все модели наследуются от общего **`Base`** (иначе не попадут в registry/метаданные).
  - `Base` — PK типа **UUID**; `BaseInt` — PK типа **int/bigint**.
- Миксин **`TimestampMixin`** → `created_at` + `updated_at`.

### Правила именования
- **Модель — в единственном числе** (`File`), **таблица — во множественном** (`files`).
- **PK всегда называется `id`.** Свойства: уникальность + минимальность.
- **Булевы поля всегда с префикса `is_`** (`is_active`).

### Идентификаторы (PK)
Порядок предпочтения: **UUID v7** → `int` → UUID v4.
- UUID v7 сортируется по дате как автоинкремент, но не даёт перебор по счётчику (безопасность).
- Последовательный int-id уязвим к перебору (`order_id` 1,2,3…).

### Колонки и дефолты
- Всегда задавать **`server_default`** (а не только Python `default`) — для обслуживания БД сырым SQL.
- Даты — тип `timestamp`, **всё в UTC**; конвертация в таймзону пользователя — на отображении.
- `created_at` / `updated_at` добавлять **всегда** (аудит + инкрементальная выгрузка в DWH).

### Enum
- **Не использовать нативные ENUM-типы БД** — боль с миграциями (удаление значения = 3-шаговая миграция).
- Enum описывается на стороне Python, в БД хранится строкой/int.

### Типизированные колонки, а не JSON-свалка
- **Сущность хранится типизированными колонками**, по одной на поле. Таблица вида
  `(id, created_at, data jsonb)` со всей сущностью в JSON — запрещена: JSONB не
  индексируется под поиск/фильтры/сортировку/агрегации так, как реляционная колонка.
- **`JSONB` допустим только как ДОПОЛНИТЕЛЬНОЕ поле** (`extra`/`metadata`) для
  действительно нерегулярных/разнородных данных, не как способ обойти проектирование
  схемы. Поля, по которым ищут/фильтруют/строят отчёты, — всегда отдельные колонки
  с индексами.
- Поле «созрело» из `extra` в важное для бизнеса → **промотируется в типизированную
  колонку** миграцией (expand→contract). `extra` — зал ожидания, не постоянный дом.

### Связи (relationships)
- FK: `ForeignKey("storage.id", ondelete="CASCADE")`.
- **`index=True` на FK обязателен** — FK не индексируются автоматически; без индекса — Sequential Scan.
- **`passive_deletes=True`** на relationship — удаление детей отдаётся БД (`ON DELETE CASCADE`),
  иначе SQLAlchemy грузит все связанные объекты в память и падает на больших объёмах.

---

## 9. Миграции (Alembic)

- Подход **Model First**: модели → автогенерация DDL.
- В `env.py` импортировать **все модели** (`from ... import *`, чтобы isort не сломал порядок),
  собрать `target_metadata`.
- Имя файла миграции: шаблон `год_месяц_день_<slug>`.
- Явно прокидывать **схему БД** (`DB_SCHEMA`) — `public` может быть запрещена.
- `async_fallback=true` — async с откатом на sync.
- `use_alter=true` — для разрыва циклических FK-зависимостей (связь отдельным ALTER).
- При использовании `sqlalchemy-utils` — дописать `render_item` (нестандартные типы).
- Применение: `alembic upgrade head`.
- **Expand → contract для breaking DDL.** Несовместимое изменение схемы разносится на
  два релиза: сначала **expand** (добавить новую колонку/таблицу, смигрировать данные,
  код читает и старое и новое), затем — отдельным релизом — **contract** (удалить
  старое). Схема каждого релиза совместима с кодом предыдущего — деплой и откат не
  ломают работающие реплики.

---

## 10. DI-контейнер — dishka

> Заменяет самописные **абстрактные фабрики** и использование `fastapi.Depends`
> как IoC-контейнера. Принципы инверсии зависимостей и Protocol-абстракции — сохраняются;
> меняется только механизм сборки и управления жизненным циклом.

### Что dishka берёт на себя
1. Создание объектов (репозитории, сервисы, use cases) по контракту-протоколу.
2. Связывание реализации с интерфейсом (`provides=Protocol`).
3. **Управление жизненным циклом** через scope (то, что фабрики не закрывали):
   - **`Scope.APP`** — синглтоны: engine, `async_sessionmaker`, конфиги, пулы соединений.
   - **`Scope.REQUEST`** — на запрос: `AsyncSession`, репозитории, сервисы, use cases.
4. Управление **сессией и транзакцией** — через провайдер-генератор (`yield` + commit/rollback).
5. Единая точка конфигурации провайдера хранилища (Postgres/Redis/in-memory).

### Провайдеры (пример)

```python
from dishka import Provider, Scope, provide, make_async_container
from collections.abc import AsyncIterable

class AppProvider(Provider):
    # APP-scope: создаётся один раз на приложение
    @provide(scope=Scope.APP)
    def sessionmaker(self, settings: Settings) -> async_sessionmaker[AsyncSession]:
        engine = create_async_engine(settings.db_dsn)
        return async_sessionmaker(engine, expire_on_commit=False)

    # REQUEST-scope: сессия с управлением транзакцией
    @provide(scope=Scope.REQUEST)
    async def session(
        self, maker: async_sessionmaker[AsyncSession]
    ) -> AsyncIterable[AsyncSession]:
        async with maker() as session:
            yield session            # commit/rollback по выходу из запроса

    # Привязка реализации к протоколу
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

### Интеграция с FastAPI

```python
from dishka import make_async_container
from dishka.integrations.fastapi import setup_dishka, FromDishka, inject

# bootstrap.py
container = make_async_container(AppProvider(), context={Settings: settings})
setup_dishka(container, app)
# закрытие контейнера (await container.close()) — в lifespan при остановке

# router.py — точка инъекции, только UseCase
@router.get("/users")
@inject
async def get_users(use_case: FromDishka[UsersUseCaseProtocol]) -> list[UserResponseSchema]:
    return await use_case()
```

### Правила
- Контроллер получает **только UseCase** через `FromDishka[...]`.
- Выбор реализации (Postgres/Redis/in-memory) — сменой одной привязки `provide(..., provides=...)`,
  без правок бизнес-логики.
- Контейнер инициализируется в `bootstrap.py`, закрывается в `lifespan`.
- Файл провайдеров домена — `providers.py` (вместо старого `deps.py`).

---

## 11. Обработка ошибок

### Иерархия исключений
```
Exception (Python)
└── CoreException                        # корневой базовый, в core
    ├── ModelNotFound        → 404
    ├── PermissionDenied     → 403
    ├── ModelAlreadyExists   → 409
    ├── ValidationError      → 422
    └── …                                # по одному базовому «под HTTP-смысл»
        └── <конкретные доменные ошибки> # в своих доменах, наследуют базовые
```

### Правила
- **Базовые/корневые исключения — в `core`.** Доменные — в `apps/<domain>/exceptions.py`,
  наследуют базовые.
- **До контроллера бросаются только бизнес-ошибки** (фреймворк-независимые).
  **После контроллера — только HTTP-ошибки.** Контроллер — граница превращения доменной ошибки в HTTP.
- Маппинг в HTTP — **централизованные FastAPI exception handlers**, по одному на базовый класс
  (наследники покрываются автоматически). Регистрация — единой функцией в `core.exceptions.handlers`.
- **Единый формат ответа об ошибке** (поле `error_message` + HTTP-код) — фронт знает, чего ждать.

### Именование
- **Доменные (бизнес) исключения — суффикс `Error`** (ожидаемые, «проверяемые»).
- **`Exception`** — для системных/фреймворковых исключений.
- Базовые — семантические по HTTP-смыслу (`ModelNotFound`, `PermissionDenied`, …).

---

## 12. Конфигурация (Settings)

- Класс настроек наследует **`BaseSettings`** (pydantic-settings).
- `model_config = SettingsConfigDict(env_file=".env")`.
- Значения — из `.env`; в репозитории лежит **`.env.example`** (`cp .env.example .env`).
  `.env` — в `.gitignore` и `.dockerignore`.
- Boolean-поля — явно `True`/`False`.
- Типовые поля: `DEBUG`, `BASE_URL`, `BASE_DIR`, `SECRET_KEY`, `CORS` (allow_origins),
  `DB_PROVIDER/USER/PASSWORD/HOST/PORT/NAME`, `DB_SCHEMA`. Опционально — `SENTRY_DSN`.
- Экземпляр settings создаётся в `main.py` и прокидывается в приложение и в Alembic `env.py`
  (в целевой схеме — через dishka context).

---

## 13. Общая библиотека (core / shared)

- ~80% кода повторяется между сервисами → выносится в общую библиотеку («ядро»).
- Подключается **отдельным пакетом** (PyPI), версионируется **semantic-release**.
- Содержит:
  - **Репозитории** (protocol + несколько реализаций): cache (in-memory/Redis),
    DB (in-memory/Postgres), file storage (S3/MinIO — **по умолчанию S3**), settings-repository.
  - **Базовые модели**: `Base` (UUID PK), `BaseInt` (int PK), миксины, naming conventions,
    async engine, фабрика сессий.
  - **Базовые исключения** + регистрация handlers.
  - **Сервисы**: криптография, распределённый Lock (Redis), transaction service, seed/fixtures.
  - **Схемы**: базовые request/response, `StatusResponse`.
  - **Utils**: пулы соединений — **singleton** (иначе пул поднимается на каждый запрос).
  - **DI-провайдеры** (dishka) и общий healthcheck.
- Пулы соединений (DB/thread) создавать как **singleton**.

---

## 14. Коммиты и версионирование

### Conventional Commits
```
тип[необязательный контекст]: описание

[тело]
[footer]
```
Типы: **`feat`** (новый функционал), **`fix`** (багфикс), плюс `build`, `chore`, `ci`,
`docs`, `style`, `refactor`, `perf`, `test`.

### Breaking changes
- Отмечается `!` после типа (`feat!: …`) **или** footer `BREAKING CHANGE:`.

### SemVer (MAJOR.MINOR.PATCH)
- **MAJOR** — breaking change (сломана обратная совместимость).
- **MINOR** — `feat` (новый функционал без слома).
- **PATCH** — `fix` + всё остальное (`docs`, `chore`, `style`, …).
- Build-сегмент через дефис (`10.8.5-1`) — для собственных Docker-registry.

### Автоматизация
- **semantic-release**: при пуше в `main` анализирует коммиты, вычисляет версию,
  релизит образ/библиотеку. Коммит бампа версии — с `[skip ci]`.
- → **Осмысленные коммиты по соглашению обязательны** (от них зависит расчёт версии).

---

## 15. Качество кода

- **Pyright strict** (`python.analysis.typeCheckingMode = strict`) + **mypy**.
- **Ruff** (линт, автофикс) + **Black** (формат).
- **isort**: порядок секций future → stdlib → third-party → first-party → local-folder.
- **pre-commit** (`pre-commit install`): хуки перед каждым коммитом — case/merge-conflict,
  end-of-file/whitespace, black, ruff, проверка типов. Прогон всех: `pre-commit run --all-files`.
- **pytest**: `asyncio_mode = auto`; в pre-commit тесты обычно закомментированы (идут в CI), coverage.
- `assert` использовать свободно — под `python -O` они вырезаются, прод не замедляют.

---

## 16. Инфраструктура (docker-compose)

Сервисы поднимаются через docker-compose; приложение подключается по **именам сервисов как хостам**.

- **PostgreSQL** — порт `5432`, env `DB_USER/PASSWORD/NAME`, **healthcheck**.
- **Redis** — порт `6379`, кэш, **healthcheck**.
- **MinIO** (S3-совместимое, опционально) — API `9000`, консоль `9001`;
  отдельный `mc`-job создаёт bucket. Доступ к файлам в проде — **через своё приложение**, не анонимно.

Правило: **к каждому сервису добавлять healthcheck** (приложение ждёт готовности зависимостей).

---

## 17. CI/CD и релиз

GitHub Actions:
- **На pull request**: линтинг (+ тесты).
- **На push в `main`**: линтинг + тесты + релиз.

Шаги релиза:
1. `checkout`.
2. Установка **uv** (action), `uv sync`.
3. **semantic-release** считает версию (конфиг `[tool.semantic_release]` в `pyproject.toml`,
   `version_toml` → путь к версии; commit `[skip ci]`).
4. `upload_to_pypi=false`, `upload_to_release=true` — semantic-release делает GitHub Release.
5. **build & publish** — отдельным шагом, только если `outputs.released == 'true'`;
   для PyPI нужен токен (секрет).

---

## 18. Чеклист нового backend-сервиса

0. **Спроектировать схему БД** до кода (таблицы, FK, `created_at`/`updated_at`,
   мягкое удаление `deleted_at`).
1. Создать **пустой** репозиторий на GitHub (без файлов).
2. Склонировать boilerplate, перепривязать remote:
   `git remote add origin <url>` → `git branch -M main` → `init`-коммит → `git push -u origin main`.
3. Переименовать `src` в имя проекта; открыть в редакторе.
4. `cp .env.example .env`, заполнить (Postgres/Redis/S3/секреты).
5. Поднять docker-compose, проверить подключения (Postgres, Redis, MinIO + создать бакеты `*` и `*-test`).
6. `uv sync`, выбрать интерпретатор `.venv`, проверить CLI (`python -m cli`).
7. Собрать `main.py`/`bootstrap.py`: метаданные из `pyproject.toml` → FastAPI(title/version),
   middleware (CORS, host), **инициализация dishka-контейнера**.
8. Запустить (`uv run uvicorn`, порт 8000), проверить `/docs` (Swagger) и `/health`.

---

## Сводка ключевых «do / don't»

**DO**
- uv, FastAPI, SQLAlchemy 2.0 как query builder, Pydantic v2, dishka, Alembic.
- Чистая архитектура: зависеть от `Protocol`, не от реализации; иерархия строго вниз.
- Изоляция доменов: чужой домен — только через публичный протокол или событие;
  `import-linter` в CI.
- Транзакция: дефолт — на запрос (провайдер); несколько репозиториев — UoW; репозиторий
  не коммитит.
- Сложные чтения — read-only query-сервис; expand→contract для breaking DDL.
- DTO между слоями; ORM не выходит из репозитория.
- PK `id` (UUID v7), модель — ед. число, таблица — мн. число, булевы — `is_*`.
- `server_default`, UTC, `created_at`/`updated_at`, `index=True` на FK, `passive_deletes=True`.
- Async + явная загрузка связей (`selectinload`, `lazy="raise"`).
- Бизнес-ошибки до контроллера, HTTP-ошибки после; централизованные handlers; единый формат.
- Conventional Commits + semantic-release; pre-commit + Pyright strict.
- Healthcheck на каждый сервис и эндпоинт.

**DON'T**
- pip / poetry; Django (кроме legacy); SQLAlchemy как ORM.
- Импортировать внутренности чужого домена; `commit` в репозитории; breaking DDL одним
  релизом с кодом.
- Нативные ENUM в БД; lazy loading в async; int-id там, где важна безопасность.
- ORM-объекты выше репозитория; бизнес-логика в контроллере.
- Самописные DI-фабрики / `Depends`-как-IoC — только **dishka**.
- Скрипты в корне проекта (только Typer CLI).
