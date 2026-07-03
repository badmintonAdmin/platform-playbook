---
name: new-domain
description: Создать новый backend-домен (bounded context) по эталонной структуре платформы — все слои, протоколы, dishka-провайдеры, регистрация, миграция, тесты. Use when the user asks to add a new backend domain / module / entity with CRUD ("новый домен", "добавь сущность X", "new domain/module").
---

# Новый backend-домен

Эталон: [rules/BACKEND_STRUCTURE.md](../../../rules/BACKEND_STRUCTURE.md) (обязательно
прочитать перед началом). Законы: [rules/BACKEND_RULES.md](../../../rules/BACKEND_RULES.md) §3–§10.

## Шаг 0 — согласовать с пользователем
- **Имя домена** — мн. число, snake_case (`orders`); **сущность** — ед. число (`Order`).
- Состав полей модели, связи с другими доменами, какие операции нужны (полный CRUD или часть).
- Если домен зависит от другого домена — только через его публичный протокол или событие
  (правила 41–44), НИКОГДА через импорт чужих internals.

## Шаг 1 — создать структуру

```
apps/<domain>/
├── router.py  ├── schemas.py  ├── use_case.py  ├── services.py
├── repositories.py  ├── models.py  ├── exceptions.py  └── providers.py
```

Простой домен — файлы; **вторая сущность в домене → слой разворачивается в папку с
файлом на сущность; use case — файл на user story; файл ≤ 400 строк (CI-гейт);
свалки `utils.py` в домене запрещены** (правила 46–48). `events.py` — только
если домен публикует события (тогда → скилл `new-event`).

## Шаг 2 — писать в порядке снизу вверх
1. **`models.py`** — наследовать `Base` (+`TimestampMixin`); таблица мн. число, PK `id`
   (UUID v7), булевы `is_*`, `server_default`, `index=True` на FK, `lazy="raise"`.
2. **Импортировать модель в `migrations/env.py`** — иначе autogenerate её не увидит.
3. **`schemas.py`** — Read/Create/Update (граница репозитория) + Request/Response
   (граница use case); `camelCase` наружу, `extra="forbid"`.
4. **`repositories.py`** — `Protocol` + `Impl(GenericRepository)`; DTO in / DTO out;
   commit НЕ делает (только flush).
5. **`services.py`** — `Protocol` + `Impl`; инжектит протоколы.
6. **`use_case.py`** — по одному callable-классу на user story; возвращает Response-схему;
   проверка прав/владения — здесь (BOLA, правило 82).
7. **`providers.py`** — все привязки `provide(Impl, provides=Protocol, scope=Scope.REQUEST)`.
8. **`router.py`** — `@inject` + `FromDishka[UseCaseProtocol]`, без бизнес-логики.

## Шаг 3 — регистрация (обязательно оба)
1. Роутер → в агрегатор роутеров.
2. `<Domain>Provider` → в `make_async_container(...)` в `bootstrap.py`.

## Шаг 4 — миграция и тесты
- `uv run alembic revision --autogenerate -m "..."` → проверить сгенерированный DDL глазами.
- Тесты зеркалят `src/`: unit на use case/service (in-memory репозиторий через протокол),
  интеграционный на репозиторий.

## Верификация перед завершением
- [ ] import-linter / typecheck (pyright) / тесты проходят
- [ ] Домен не импортирует internals других доменов (правила 42–44)
- [ ] ORM не выходит из репозитория; репозиторий не коммитит (правила 25, 27)
- [ ] Роутер и Provider зарегистрированы; `/docs` показывает эндпоинты
- [ ] Сверить diff по разделам «Слои», «DI», «Домены», «Модели БД» из
      [rules/RULES_CHECKLIST.md](../../../rules/RULES_CHECKLIST.md)
