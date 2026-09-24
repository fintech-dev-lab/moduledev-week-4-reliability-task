# Контракт наблюдаемости недели 4

## Health

Каждый response health имеет `Content-Type: application/json` и минимальную форму:

```json
{"status":"live"}
```

или:

```json
{"status":"ready"}
```

Успешный probe возвращает HTTP 200. Неуспешный readiness возвращает HTTP 503 и `{"status":"not_ready","code":"dependency.unavailable"}`. Liveness не выполняет глубокую проверку PostgreSQL/provider и остаётся 200, пока process способен обслужить probe.

PostgreSQL является readiness dependency для API, worker, dispatcher и reconciler. Gateway является dependency adapter. Provider не является readiness dependency API, worker или dispatcher.

## OpenMetrics

`GET /metrics` возвращает HTTP 200, media type `application/openmetrics-text; version=1.0.0; charset=utf-8` и document с завершающим `# EOF`.

`api` публикует как минимум:

| Metric | Type | Значение |
|---|---|---|
| `workflow_jobs_ready` | gauge | Jobs, доступные для claim сейчас |
| `workflow_job_oldest_age_seconds` | gauge | Возраст старейшего доступного job; 0 при отсутствии |
| `workflow_processes_waiting` | gauge | Processes в WAITING_SIGNAL/WAITING_MANUAL |
| `outbox_pending` | gauge | Число deliveries в `PENDING`, `LEASED`, `RETRY_WAIT`, включая отложенный retry |
| `outbox_oldest_age_seconds` | gauge | Возраст от `created_at` старейшего delivery в тех же трёх состояниях; 0 при отсутствии |
| `workflow_failures_total` | counter | Число зафиксированных terminal workflow failures |

`DEAD`, `DELIVERED` и `CONFIRMED` не входят в обе Outbox-метрики. Ожидающий квитанцию процесс после `DEAD` по-прежнему входит в `workflow_processes_waiting`.

В OpenMetrics counter объявляется по имени семейства, без суффикса `_total`; суффикс есть у sample:

```text
# TYPE workflow_failures counter
workflow_failures_total 0
# EOF
```

Дополнительные labels имеют ограниченный набор значений. Запрещены label names `correlation_id`, `request_id`, `operation_id`, `process_id`, `step_instance_id`, `job_id`, `execution_id`, `attempt_id`, `outbox_id`, `external_request_id`, `message_id`, `decision_id` и их camelCase-варианты. Значения labels не содержат UUID/unique identifiers, имеют длину не более 64 символов; одна scrape не содержит более 20 значений одного label.

## `diagnostics.trace`

Action manifest version 1:

| Поле | Значение |
|---|---|
| module/action | `diagnostics.trace` |
| method | `POST` |
| required policy | `diagnostics:read` |
| idempotency mode/scope | `none` / `none` |
| outcomes | `FOUND` |
| request schema | `diagnostics-trace.payload.schema.json` |
| response schema | `diagnostics-trace.result.schema.json` |

Поддерживаются identifiers: `correlationId`, `requestId`, `operationId`, `processId`, `stepInstanceId`, `jobId`, `executionId`, `attemptId`, `externalRequestId`, `messageId`, `decisionId`.

Все известные identifiers одной цепочки возвращают один и тот же набор предметных фактов; отличается только `query.identifier` и `query.matchedBy`. Массивы упорядочиваются по времени, затем по identifier для детерминированного результата. Поля используют camelCase, timestamps сериализуются как RFC 3339 UTC, amount как строка с двумя десятичными знаками.

Неизвестный identifier возвращает HTTP 404 с code `diagnostics.trace_not_found`. Невалидная форма payload возвращает `422 payload.invalid`. Trace не содержит JWT, signature, secret, полные payload/body, provider callback `message` и manual `reason`.

## `diagnostics.stalled`

Action возвращает операции с затянувшимся ожиданием квитанции после исчерпания автоматических попыток доставки.

| Поле | Значение |
|---|---|
| module/action/version | `diagnostics.stalled`, version 1 |
| method/route | `POST /api/diagnostics/stalled` |
| required policy | `diagnostics:read` |
| idempotency mode/scope | `none` / `none` |
| outcomes | `FOUND` |
| enabled/default | `true` / `true` |
| payload | `{}` |
| request schema | `diagnostics-stalled.payload.schema.json` |
| response schema | `diagnostics-stalled.result.schema.json` |

Ответ — HTTP 200, стандартный envelope `status=ok`, `outcome=FOUND`, `result={"items":[...]}` и correlation metadata. Каждый элемент содержит только `operationId` и `processId` типа UUID и непустой строковый `externalRequestId`. Одна операция встречается не более одного раза; порядок — по `operationId` в канонической UUID-записи. Возвращается вся выборка без пагинации; пустая выборка — `{"items":[]}` с теми же HTTP status и outcome.

Любое дополнительное поле payload возвращает `422 payload.invalid`. Отсутствующий или невалидный JWT — HTTP 401; JWT без `diagnostics:read` — HTTP 403. Клиент не передаёт SQL target или текст запроса. Action регистрируется обычным manifest и выполняется PostgreSQL-функцией через generic runtime. Он читает актуальное сохранённое состояние, не меняет предметные данные, не создаёт задания и не повторяет доставку. Обычные access logs и action audit допустимы. Результат не содержит payload, реквизитов, секретов или текстов сообщений.

## Structured logs

Каждая строка application logs является отдельным JSON object. События выполнения содержат применимые IDs, action/flow/step, outcome или safe error code и duration. Уникальные IDs допустимы в logs, но не в metric labels.

Failpoint acknowledgement имеет точную форму:

```json
{"event":"failpoint.reached","name":"after_job_claim","instanceId":"<non-empty>"}
```

Дополнительные поля допустимы. JWT, HMAC secret/signature, database passwords, полный payment payload, receipt body, callback `message` и manual `reason` запрещены.
