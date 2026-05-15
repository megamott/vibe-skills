---
paths:
  - "tests/**/*.py"
---

# Стиль тестов проекта Search

## Общие принципы

- Паттерн **AAA**: Arrange (подготовка) → Act (действие) → Assert (проверка)
- Все тесты **async** (pytest-asyncio)
- Все переменные аннотируются `typing.Final`
- Все функции возвращают `-> None`
- Предпочитать **интеграционные тесты** юнит-тестам; юнит-тесты — для доменных агрегатов с нетривиальной логикой

## Генерация данных

- Использовать faker для генерации тестовых данных, не хардкод
- `FAKER_OBJ: typing.Final = faker.Faker("ru_RU")` в начале файла
- Использовать polyfactory для создания тестовых объектов через фабрики из `tests/factories.py`

## Параметризация

- `@pytest.mark.parametrize` вместо дублирования тестов
- Прогонять код по разным путям в нескольких итерациях

## Именование тестов

- `test_<что_тестируется>_<условие>` — например: `test_update_documents_with_rag_index_regenerates_chunks`
- Для параметризованных: суффикс `_parametrized`

## Мокирование — обязательное правило

**Всегда используй `monkeypatch` (pytest fixture) вместо `unittest.mock.patch`.**

```python
# ✅ ПРАВИЛЬНО
async def test_something(monkeypatch: pytest.MonkeyPatch) -> None:
    def _raise_error(_self: object) -> None:
        raise SomeError("reason")

    monkeypatch.setattr("module.Class.method", _raise_error)

# ❌ ЗАПРЕЩЕНО
async def test_something() -> None:
    with unittest.mock.patch("module.Class.method", side_effect=SomeError("reason")):
        ...
```

`monkeypatch` автоматически откатывает изменения, совместим с `autouse`-фикстурами и не требует `import unittest.mock`. Для возврата значения — передай лямбду или функцию. Для выброса исключения — определи вспомогательную функцию внутри теста.

## Мокирование OpenSearch

- `AsyncOpenSearchMock` из `tests/infrastructure/mocks/opensearch_mocks.py` — заменяет реальный клиент через DI-контейнер
- `OpenSearchCustomRequestsMock` имитирует `/_msearch/template` запросы и возвращает сгенерированные hit-данные
- `generate_opensearch_msearch_response` — фабричная функция для создания ответа msearch по списку индексов

## AsyncMock

- Проверять через `await_count` / `await_args_list`, **НЕ** `call_count` / `call_args_list`

## Моки проекта

- Все моки в `tests/infrastructure/mocks/`:
  - `AsyncOpenSearchMock` — мок клиента OpenSearch (msearch, get, search)
  - `RerankerMock` — мок Cohere reranker клиента (`reranker_mocks.py`)
  - `HttpxClientMock` — мок httpx-клиента для внешних HTTP-вызовов (`httpx_client_mock.py`)

## Фабрики

- Используй фабрики из `tests/factories.py`:
  ```python
  source: typing.Final = factories.SourceFactory.build()
  index: typing.Final = factories.IndexFactory.build(is_rag=True)
  ```

## Исключения в тестах

```python
with pytest.raises(SomeException, match=re.escape("Expected error message")):
    some_function_that_raises()
```

## Тесты должны быть параллелизируемы

- Избегать зависимостей между тестами и общего изменяемого состояния
- Совместимость с pytest-xdist (`-n auto`)

## Что тестировать

- Все публичные методы доменных агрегатов
- Обработчики команд и запросов (happy path + edge cases)
- Валидацию входных данных и граничные случаи
- Обработку ошибок и исключений
- Логику поиска: гибридный поиск, фильтрацию, пагинацию
- Обогащение результатов заявками (паттерн `XX-NNNNNNN`)
- RAG-контекст с ранжированием Cohere
- Историю поиска (сохранение, получение, скрытие)

## Что НЕ тестировать

- Приватные методы (тестировать через публичные)
- Простые геттеры/сеттеры без логики
- Код сторонних библиотек
