# Зона 4. Аварийный режим

## Срок и сдача

Работа выполняется с 24 по 30 сентября 2026 года. Дедлайн — **30 сентября, 23:59 МСК**.

До дедлайна отправьте куратору URL прежнего репозитория решения, ветку и полный SHA commit; заранее предоставьте доступ. Секреты, build output, логи и сгенерированные отчёты в сдачу не включаются.

## Результат

Учебный зачёт подтверждает обязательные условия результата недели 3 и сохранение
результата при конкурентных исполнителях, stale completion и аварийных границах commit.
Полное техническое соответствие и баллы считаются отдельно. Частные форматы остаются
замечаниями; непроверенное критическое свойство даёт «Недостаточно данных».
Полный [регламент зачёта](../docs/review-calibration-weeks-3-4.md).

Продолжите тот же репозиторий, который сдавали на неделях 1–3. Не переносите решение в новый starter и не меняйте архитектурные границы предыдущих недель.

```text
command -> PostgreSQL state + Outbox
Outbox -> two Python dispatchers -> provider v0.2.0
callback -> adapter -> generic C# API -> Inbox
Inbox -> two Python reconcilers -> workflow signal -> worker
persisted facts -> metrics + diagnostics.trace + diagnostics.stalled
```

Результат недели: после контролируемого сбоя и полного restart нет потерянного сообщения, второго предметного эффекта или ложного успеха. Recovery выполняют обычные worker/dispatcher/reconciler loops по состоянию PostgreSQL, а не ручной SQL.

## Compose seam

Готовность финальной сдачи включает чистый запуск, полный README/runbook и одну команду
проверки. Недокументированное вмешательство проверяющего является дефектом готовности.
Устранение ранее доставленных замечаний подтверждается по коду и сохранённым сценариям;
повтор не предполагается только по похожему названию ошибки.

Сохраните все service names недели 3 и добавьте два постоянно работающих сервиса:

| Service | Контракт |
|---|---|
| `outbox-dispatcher-b` | Второй экземпляр dispatcher, owner `outbox-dispatcher-b` |
| `inbox-reconciler-b` | Второй экземпляр reconciler |

Все пять Python services используют один локально собранный Python 3.12+ image с разными entrypoints. Вторые экземпляры не получают новый image, отдельное хранилище или дополнительные DB grants. Host-порт публикует только `gateway`.

## Надёжная доставка

Один claim выполняется короткой транзакцией. Provider HTTP выполняется после commit и завершается условно по owner/`leaseVersion`. Просроченная попытка не может перезаписать результат нового владельца.

Требуются:

- bounded retry только для transport error, timeout, HTTP 408/429/5xx;
- external timeout, `nextAttemptAt`, exponential backoff и jitter;
- стабильные body, `externalRequestId` и correlation identifier на всех attempts;
- reclaim просроченного delivery;
- `DEAD` после исчерпания attempts;
- поздняя валидная receipt переводит delivery в `CONFIRMED` и продолжает process.

`DEAD` является техническим исходом доставки. Он не переводит operation в `REJECTED` и ожидающий receipt process в `FAILED`.

Диагностика исчерпания хранится в Outbox: `state = DEAD`, непустые `dead_at` и `last_error_code`, сохранённый `attempt_count`. В trace это `state`, `deadAt`, `lastErrorCode`, `attemptCount`. Отдельный флаг ошибки у process не требуется.

Test profile: четыре attempts, включая первую; timeout 500 ms; base delays 200/400/800 ms; jitter 0–100 ms; delivery/job lease 2 s; dispatcher/worker poll не более 100 ms.

## Workflow recovery

- Expired job повторно захватывается с новым `leaseVersion`.
- Stale completion возвращает `workflow.lease_stale` и не меняет state.
- Retry сохраняет `jobId`/`executionId`, создаёт новый `attemptId`.
- READY, RETRY_WAIT, WAITING_SIGNAL и WAITING_MANUAL переживают restart.
- Action retry не повторяет предметный effect.
- Исчерпание action retries даёт job `DEAD`, process `FAILED` и safe error code.
- Callback и manual decision после полного restart остаются идемпотентными.

## Health и OpenMetrics

Все перечисленные процессы предоставляют `GET /health/live`, `GET /health/ready`, `GET /metrics` на внутренних адресах:

| Процесс | Адрес |
|---|---|
| `api` | `http://api:8080` |
| `worker-a`, `worker-b` | `http://<service>:8080` |
| оба dispatcher | `http://<service>:8080` |
| `receipt-adapter` | `http://receipt-adapter:8082` |
| оба reconciler | `http://<service>:8080` |

PostgreSQL является readiness dependency API, worker, dispatcher и reconciler. Adapter зависит от gateway. Provider outage не делает API, worker или dispatcher `not ready`.

`api:8080/metrics` публикует обязательные series:

```text
workflow_jobs_ready
workflow_job_oldest_age_seconds
workflow_processes_waiting
outbox_pending
outbox_oldest_age_seconds
workflow_failures_total
```

Остальные `/metrics` также возвращают корректный OpenMetrics document. Уникальные IDs запрещены как names или values labels; value не длиннее 64 символов, одна scrape содержит не более 20 значений одного label.

## `diagnostics.trace`

Зарегистрируйте action `diagnostics.trace` version 1:

- policy `diagnostics:read`;
- idempotency mode `none`;
- outcome `FOUND`;
- payload `{"identifier":"<value>"}`;
- неизвестный identifier: HTTP 404, code `diagnostics.trace_not_found`.

Trace связывает dispatch, operation/events, pinned process, steps, jobs/attempts, Outbox/Inbox и receipt/decision. Поддерживаемые identifiers и точная response schema опубликованы в [контракте наблюдаемости](../docs/observability-contracts.md).

## `diagnostics.stalled`

Реализуйте action `diagnostics.stalled`, возвращающий операции с затянувшимся ожиданием квитанции после исчерпания автоматических попыток доставки.

Version 1 публикуется как `POST /api/diagnostics/stalled`, требует `diagnostics:read`, принимает `{}` и не требует idempotency key. Ответ: HTTP 200, outcome `FOUND`, result `{"items":[...]}`. Каждый элемент содержит `operationId`, `processId`, `externalRequestId`; операции не повторяются и упорядочены по `operationId`. Пустая выборка — `{"items":[]}`. Action читает PostgreSQL через тот же generic runtime и не меняет предметное состояние.

Точные схемы, ошибки и правила формата — в [контракте наблюдаемости](../docs/observability-contracts.md).

## Failpoints

При `COURSE_TEST_PROFILE=1` компонент принимает одну точку из `COURSE_FAILPOINT`, пишет одну JSON-строку `{"event":"failpoint.reached","name":"<name>","instanceId":"<id>"}` и блокируется до остановки:

- `after_job_claim`;
- `after_action_before_finish`;
- `after_outbox_claim`;
- `after_provider_response`;
- `after_inbox_saved`;
- `after_manual_decision`.

Production profile игнорирует failpoints и не предоставляет endpoint для их активации. Сервисы, границы commit и порядок остановки конкурирующих исполнителей перечислены в [таблице failpoints](../docs/07-autocheck-outline.md#детерминированные-failpoints).

## README и runbook

README содержит требования к окружению, входную конфигурацию, команды первого запуска и полной проверки, признаки readiness и безопасную диагностику типовых отказов.

В runbook приведите команды вызова `diagnostics.trace` и `diagnostics.stalled` и чтения их результатов.

## Acceptance

- Два dispatcher/reconciler используют один local Python image и прежние least-privilege roles.
- Provider outage сохраняет Outbox; восстановление автоматически продолжает delivery.
- Два dispatcher и lost response не создают второй provider payment.
- `DEAD` не создаёт ложный domain result; late receipt продолжает process.
- Jobs/deliveries восстанавливаются после lease expiry, stale finish отклоняется.
- Все шесть failpoints дают acknowledgement и recovery без ручного DML.
- Полный restart сохраняет state и идемпотентность callback/manual decision.
- PostgreSQL state переживает пересоздание Compose project благодаря declared named volume.
- Live/ready имеют разную семантику, metrics совпадают с очередями.
- Trace по известному identifier содержит полное сохранённое evidence.
- `diagnostics.stalled` возвращает выборку операций в опубликованном формате без изменения предметных данных.
- Secrets/full payload отсутствуют в repo, image layers, logs, trace и report.

Действующая редакция задания — `week-4.1`. Полный контракт, configuration seam и public checker опубликованы в [docs](../docs/README.md).
