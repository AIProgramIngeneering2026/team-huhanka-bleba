# Контракт Tool

Tool — один шаг конвейера отчёта: собрать данные из Grafana, посчитать итоги, собрать PDF, доставить в один канал.
Dispatcher не знает конкретных Tool: он берёт Tool из `ToolRegistry` по имени шага, вызывает `ExecuteAsync`
и превращает результат в переход Job. Новый Tool добавляется регистрацией в DI, без правки Dispatcher.

Место в архитектуре — [architecture.md](architecture.md#схема-компонентов).

## Интерфейс

Интерфейс живёт в проекте `Domain` и не зависит от ASP.NET, EF Core и Hangfire.

```csharp
namespace GrafanaReports.Domain.Tools;

public interface ITool
{
    /// Имя шага, по которому Registry находит Tool: "grafana.collect", "delivery.minio".
    string Name { get; }

    /// Проверка параметров шага. Не ходит во внешние системы, не бросает исключений.
    /// Пустой список — параметры верны.
    IReadOnlyList<FieldError> Validate(JsonElement parameters);

    /// Выполнение шага. Ожидаемые ошибки возвращаются как ToolResult.Failure, а не исключением.
    Task<ToolResult> ExecuteAsync(ToolRequest request, CancellationToken cancellationToken);
}

public sealed record ToolRequest(
    JobId JobId,
    ReportId ReportId,
    Period Period,                 // полуинтервал [from, to) в часовом поясе отчёта
    int Attempt,                   // 1, 2, 3… — номер попытки этого шага
    JsonElement Parameters,        // параметры шага из настройки отчёта
    ArtifactRef? Input);           // результат предыдущего шага; null для первого шага

public abstract record ToolResult
{
    public sealed record Success(ArtifactRef Output) : ToolResult;
    public sealed record Failure(ToolError Error) : ToolResult;
}

public sealed record ToolError(
    ToolErrorKind Kind,
    string Code,                   // машиночитаемый код, попадает в Job.error.code
    string Message,                // для человека, без секретов и токенов
    string? Field = null,          // какой параметр не так — для InvalidParameters
    TimeSpan? RetryAfter = null);  // подсказка Dispatcher-у для Transient

public enum ToolErrorKind
{
    InvalidParameters,   // параметры неверны — повтор не поможет
    Transient,           // временная ошибка внешней системы — повтор может помочь
    Permanent            // неустранимая ошибка — повтор не поможет
}

public sealed record FieldError(string Field, string Code, string Message);

public interface IToolRegistry
{
    /// Бросает UnknownOperationException (код unknown_operation), если Tool с таким именем нет.
    ITool Get(string name);
}
```

`ArtifactRef` — ссылка на промежуточный результат в `ArtifactStore` (идентификатор, тип, размер, SHA-256).
Tool читает вход и пишет выход через `IArtifactStore`; большие данные между шагами не передаются через очередь.

## Что Tool получает

| Поле                | Откуда                                  | Что гарантирует вызывающий                               |
|---------------------|-----------------------------------------|----------------------------------------------------------|
| `JobId`, `ReportId` | Job                                     | Существуют; Job сейчас в состоянии этого шага            |
| `Period`            | Job                                     | `from < to`, длина не больше 366 дней, `to` не в будущем |
| `Attempt`           | Job                                     | Начинается с 1, растёт при каждом повторе шага           |
| `Parameters`        | Настройка отчёта на момент создания Job | **Ничего**: проверяет сам Tool (см. ниже)                |
| `Input`             | Выход предыдущего шага                  | Есть в `ArtifactStore`, тип соответствует шагу           |
| `CancellationToken` | Dispatcher                              | Срабатывает при отмене Job и по таймауту шага            |

## Что Tool возвращает

- **`Success(Output)`** — шаг сделан, результат сохранён в `ArtifactStore`.
- **`Failure(ToolError)`** — шаг не сделан, вид ошибки определяет, что сделает Dispatcher.

Tool **не меняет состояние Job** и не пишет в `job_events`: это делает Dispatcher по результату.

## Какими ошибками отказывает и что делает Dispatcher

| Ответ Tool                                    | Пример `Code`                                                                      | Переход Job                                                                         |
|-----------------------------------------------|------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------|
| `Success`                                     | —                                                                                  | Следующий шаг по таблице переходов                                                  |
| `Failure(InvalidParameters)`                  | `invalid_parameters` + `Field`                                                     | `Failed` сразу, без повторов                                                        |
| `Failure(Transient)`                          | `upstream_unavailable`, `storage_unavailable`, `smtp_unavailable`                  | `Retrying`, повтор через `max(RetryAfter, backoff)`; после `maxAttempts` — `Failed` |
| `Failure(Permanent)`                          | `upstream_rejected`, `panel_not_found`, `upstream_bad_response`, `input_too_large` | `Failed` сразу, владельцу и оператору — уведомление                                 |
| `OperationCanceledException` при отмене Job   | —                                                                                  | `Cancelled` (если переход разрешён из текущего состояния)                           |
| `OperationCanceledException` по таймауту шага | —                                                                                  | Как `Transient` с кодом `step_timeout`                                              |
| Любое другое исключение                       | —                                                                                  | Как `Permanent` с кодом `tool_crashed`, стек — только в лог                         |

Повторы — на каждый шаг отдельно, `maxAttempts = 6` (первая попытка + 5 повторов) — настраивается.
Задержки перед повторами: 30 с, 2 мин, 8 мин, 15 мин, 15 мин (экспоненциально ×4, потолок 15 мин, джиттер ±20 %).
Окно повторов — около 40 мин (не меньше ~32 мин с учётом джиттера и без таймаутов шага): этого хватает, чтобы
пережить перезапуск или обновление Grafana, MinIO или SMTP, а затяжной сбой всё равно станет `Failed` с уведомлением
в пределах часа (SC-5). Если `RetryAfter` больше задержки, ждём `RetryAfter`, но не дольше потолка в 15 мин.

## Кто проверяет параметры

**Параметры проверяет Tool** — методом `Validate`, потому что только он знает, какие параметры ему нужны.
Вызывающие проверяют только своё:

1. **При сохранении отчёта** (`POST /reports`, `PUT /reports/{id}`) ReportService вызывает `Validate` у каждого
   Tool конвейера и возвращает все ошибки разом как `400 validation_failed` с полями. Так ошибка видна сразу,
   а не через неделю при плановом запуске.
2. **Перед выполнением** Dispatcher снова вызывает `Validate`: настройка могла сохраниться до изменения Tool.
   Ошибки — `Failure(InvalidParameters)`, Job → `Failed`.
3. **Dispatcher** сам параметры не разбирает. Он проверяет только, что Tool с таким именем есть в Registry
   (иначе `unknown_operation`, Job → `Failed`), и что Job в состоянии этого шага.

`Validate` проверяет форму (типы, диапазоны, обязательность). Проверки, которым нужна внешняя система
(есть ли панель на дашборде), делает `ExecuteAsync` и отказывает `Permanent`.

## Обязательства Tool

- **Идемпотентность.** Hangfire доставляет шаг «хотя бы один раз», поэтому повторный вызов с тем же `JobId`
  не должен давать второй побочный эффект (см. каждый Tool ниже).
- **Отмена.** Tool передаёт `CancellationToken` во все внешние вызовы и не глотает `OperationCanceledException`.
- **Без секретов в ошибках.** `Message` не содержит токенов, паролей, URL с подписью.
- **Лимиты.** Tool сам ограничивает объём входа и выхода и отказывает `input_too_large`, а не падает по памяти.

## Tool проекта

| Имя                | Шаг Job       | Вход             | Выход            | Внешняя система |
|--------------------|---------------|------------------|------------------|-----------------|
| `grafana.collect`  | `Collecting`  | —                | `CollectedData`  | Grafana         |
| `report.aggregate` | `Aggregating` | `CollectedData`  | `AggregatedData` | —               |
| `report.render`    | `Rendering`   | `AggregatedData` | `RenderedPdf`    | —               |
| `delivery.minio`   | `Delivering`  | `RenderedPdf`    | `StoredObject`   | MinIO           |
| `delivery.email`   | `Delivering`  | `RenderedPdf`    | `SentMail`       | SMTP            |

На шаге `Delivering` Dispatcher вызывает Tool каждого включённого канала отдельно и хранит статус канала в Job.
Повтор — только для канала со статусом `failed` (SC-6). Итог: все `delivered` → `Delivered`;
часть — `PartiallyDelivered`; ни одного — `Failed`.

### `grafana.collect` — сбор данных панелей

Берёт описание дашборда и данные выбранных панелей за период и за такой же предыдущий период (для сравнения).

**Параметры:**

```json
{
  "dashboardUid": "team-sla",
  "panelIds": [2, 4, 7],
  "variables": {"env": "prod", "service": ["api", "worker"]},
  "timeZone": "Asia/Yekaterinburg"
}
```

**`Validate`:**

| Поле           | Правило                                                                 | Код                                                        |
|----------------|-------------------------------------------------------------------------|------------------------------------------------------------|
| `dashboardUid` | обязательно, `^[a-zA-Z0-9_-]{1,40}$`                                    | `required`, `invalid_format`                               |
| `panelIds`     | обязательно, 1–50 уникальных целых `> 0`                                | `required`, `too_many_items`, `not_unique`, `invalid_type` |
| `variables`    | не больше 30 ключей, значение — строка или массив строк до 500 символов | `too_many_items`, `invalid_type`, `too_long`               |
| `timeZone`     | обязательно, известный пояс IANA                                        | `required`, `invalid_value`                                |

**Выход `CollectedData`:** по каждой панели — заголовок, единица измерения, способ свёртки из JSON панели
и временные ряды за текущий и предыдущий периоды.

**Отказы:**

| Ситуация                                                                   | Вид         | Код                                                                   |
|----------------------------------------------------------------------------|-------------|-----------------------------------------------------------------------|
| Grafana `5xx`, `429`, нет ответа за 30 с, соединение разорвано             | `Transient` | `upstream_unavailable`, `RetryAfter` из заголовка `Retry-After`       |
| Grafana `401` / `403`                                                      | `Permanent` | `upstream_rejected` — сервисный аккаунт потерял доступ                |
| Дашборд не найден (`404`)                                                  | `Permanent` | `dashboard_not_found`                                                 |
| Панели нет в JSON дашборда                                                 | `Permanent` | `panel_not_found`, `Field = panelIds[i]`                              |
| Ответ не JSON или в нём нет ожидаемых полей (`frames`, `schema`, `values`) | `Permanent` | `upstream_bad_response` — повтор не поможет, формат Grafana изменился |
| Больше 10 000 точек в ряду или больше 50 МБ данных                         | `Permanent` | `input_too_large`                                                     |

**Идемпотентность:** только читает Grafana; повторный вызов перезаписывает свой артефакт для этого `JobId`.

### `delivery.minio` — доставка в MinIO

Кладёт PDF в бакет отчётов и возвращает ключ объекта. Dispatcher записывает ключ в `report_files`
**до** перехода Job в `Delivered` (SC-2).

**Параметры:**

```json
{
  "bucket": "reports",
  "keyPrefix": "team-sla/"
}
```

**`Validate`:**

| Поле        | Правило                                                 | Код                          |
|-------------|---------------------------------------------------------|------------------------------|
| `bucket`    | обязательно, 3–63 символа, правила имён S3              | `required`, `invalid_format` |
| `keyPrefix` | необязательно, до 200 символов, без `..` и ведущего `/` | `too_long`, `invalid_format` |

**Вход:** `RenderedPdf` (ссылка на файл, размер, SHA-256).
**Выход `StoredObject`:** `objectKey = {keyPrefix}{reportId}/{period.from:yyyy-MM-dd}_{jobId}.pdf`, `etag`, `sizeBytes`.

**Отказы:**

| Ситуация                        | Вид         | Код                   |
|---------------------------------|-------------|-----------------------|
| MinIO `5xx`, нет ответа за 30 с | `Transient` | `storage_unavailable` |
| `403` / бакет не существует     | `Permanent` | `storage_rejected`    |
| PDF больше 100 МБ               | `Permanent` | `input_too_large`     |
| Входной артефакт не найден      | `Permanent` | `input_missing`       |

**Идемпотентность:** ключ объекта зависит только от отчёта, периода и `JobId`. Перед записью Tool делает `HEAD`:
если объект есть и его SHA-256 совпадает — возвращает `Success` без перезаписи (SC-6: файл в MinIO не перезаписывается
при повторе доставки).

### `delivery.email` — доставка письмом

Отправляет PDF вложением адресатам из настройки отчёта.

**Параметры:** `recipients` — 1–50 уникальных адресов (`required`, `too_many_items`, `invalid_format`);
`subjectTemplate` — до 200 символов.

**Отказы:** SMTP `4xx` или нет соединения — `Transient`, `smtp_unavailable`; SMTP `5xx` (адрес отклонён,
вложение слишком большое) — `Permanent`, `smtp_rejected`; PDF больше 20 МБ — `Permanent`, `input_too_large`.

**Идемпотентность:** `Message-ID` письма строится из `JobId`, а Dispatcher не вызывает Tool повторно для канала
со статусом `delivered`. Если SMTP принял письмо, но ответ потерялся, письмо может уйти дважды — с тем же
`Message-ID`, большинство почтовых клиентов покажут его один раз.
