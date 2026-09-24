# Неделя 4. Надёжность, восстановление и доказательство результата

## Цель

Довести платформу до состояния, в котором она выдерживает контролируемые сбои, автоматически восстанавливает незавершённую работу и позволяет доказать путь каждой операции по сохранённым данным и наблюдаемости.

## Срок и сдача

Работа выполняется с 24 по 30 сентября 2026 года. Дедлайн — 30 сентября, 23:59 МСК.

Проверяется продолжение того же репозитория участника и полный SHA commit. До дедлайна участник отправляет куратору URL, ветку и SHA и заранее предоставляет доступ. В сдачу не включаются `.env`, реальные секреты, build output, логи и сгенерированные отчёты проверки.

## Проверяемый результат

Failpoint-run останавливает API, worker или dispatcher в согласованных точках. После рестарта система продолжает обработку без потери сообщения, двойного бизнес-эффекта и ложного предметного успеха.

## Продолжение недель 1–3 и Compose seam

Неделя 4 изменяет существующее решение, а не создаёт новый сервисный каркас. Сохраняются обязательные services `gateway`, `api`, `cli`, `postgres`, `worker-a`, `worker-b`, `outbox-dispatcher`, `receipt-adapter`, `inbox-reconciler`, `provider-simulator` и добавляются:

- `outbox-dispatcher-b` — второй экземпляр того же локально собранного Python image с owner `outbox-dispatcher-b`;
- `inbox-reconciler-b` — второй экземпляр того же локально собранного Python image и той же least-privilege роли.

Оба dispatcher работают постоянно, используют разные owner и одну SQL boundary недели 3. Оба reconciler работают постоянно через `delivery.reconcile_inbox(integer)`. Новый image, отдельное хранилище или дополнительные полномочия для вторых экземпляров запрещены. Host-порт по-прежнему публикует только `gateway`.

## Надёжный Python Outbox dispatcher

Базовый Python dispatcher недели 3 необходимо расширить, не перенося в него авторитетное состояние или workflow transitions:

- конкурентным захватом записей без длительной блокировки;
- lease/fencing или эквивалентным условным завершением доставки;
- ограниченными retries;
- exponential backoff с jitter;
- `nextAttemptAt`;
- timeout внешнего вызова;
- состоянием `DEAD` после исчерпания попыток;
- восстановлением просроченной доставки;
- безопасной обработкой потери ответа.

Два Python dispatcher из одного image могут отправить запрос повторно, но стабильный `externalRequestId` и идемпотентность provider не допускают второй внешний платёж.

Поздняя валидная квитанция имеет приоритет над исчерпанием доставки: она атомарно подтверждает результат и продолжает ожидающий процесс. `DEAD` Outbox не должен ложно переводить предметную операцию в `REJECTED` или технически ожидающий процесс в `FAILED`.

Диагностический признак исчерпания — `autocheck.outbox.state = DEAD`, непустые `dead_at` и `last_error_code`, сохранённый `attempt_count`; в trace это `state`, `deadAt`, `lastErrorCode`, `attemptCount`. Process остаётся `WAITING_SIGNAL`, operation — `PROCESSING`. Отдельное поле ошибки у process не требуется.

В test profile используются четыре attempts, включая первую, provider timeout 500 ms, base delays 200/400/800 ms и добавочный jitter от 0 до 100 ms. Lease доставки и lease job равны 2 секундам, poll interval dispatcher и worker не превышает 100 ms. Все значения задаются конфигурацией; обычный профиль может использовать более консервативные интервалы.

## Восстановление workflow

Необходимо доказать:

- просроченный job повторно захватывается;
- stale completion отклоняется fencing-проверкой;
- retry сохраняет стабильный `executionId` и новый `attemptId`;
- restart worker не теряет ready/delayed/waiting jobs;
- retry action не повторяет предметный эффект;
- исчерпание action retries переводит job в `DEAD`, а процесс в `FAILED` с безопасным кодом;
- повтор callback и manual decision после полного рестарта остаётся идемпотентным.

Запрещено восстанавливать систему ручным изменением таблиц.

## Health

API, worker и participant-built Python integration services предоставляют на служебных портах:

- `GET /health/live` — процесс способен обслуживать probe;
- `GET /health/ready` — критические зависимости готовы;
- `GET /metrics` — OpenMetrics.

PostgreSQL является критической зависимостью readiness API, worker, dispatcher и reconciler. Receipt adapter зависит от gateway. Недоступность provider отражается в Outbox, retries и метриках dispatcher, но не делает API/worker `not ready`, потому что накопление Outbox — штатный режим.

Проверяемые внутренние адреса: `api:8080`, `worker-a:8080`, `worker-b:8080`, оба dispatcher на порту `8080`, `receipt-adapter:8082`, оба reconciler на порту `8080`. Эти порты доступны только в Compose network.

## Обязательные метрики

- `workflow_jobs_ready`;
- `workflow_job_oldest_age_seconds`;
- `workflow_processes_waiting`;
- `outbox_pending`;
- `outbox_oldest_age_seconds`;
- `workflow_failures_total`.

Допустимы дополнительные labels с ограниченной кардинальностью. Нельзя использовать `requestId`, `operationId`, `processId` и другие уникальные идентификаторы как names или values labels. Значение label имеет длину не более 64 символов; одна scrape содержит не более 20 значений одного label.

Все шесть обязательных серий публикует `api` как согласованную проекцию PostgreSQL. `/metrics` каждого другого проверяемого процесса также возвращает корректный OpenMetrics document; собственные component metrics остаются выбором участника.

## Структурированные логи и trace

Лог каждого шага содержит безопасный минимум:

- `correlationId`;
- `requestId` и `operationId`, если применимы;
- `processId`, flow и version;
- step key;
- `jobId`, `executionId`, `attemptId`, `leaseVersion`;
- action key/version и outcome;
- duration и error code.

Логи не содержат JWT, HMAC secret/signature, полные платёжные payload и чувствительные тела сообщений.

Action `diagnostics.trace` по любому известному сквозному идентификатору возвращает согласованную проекцию:

- HTTP/action dispatch;
- operation и events;
- process/version/steps;
- jobs и attempts;
- Outbox deliveries и Inbox messages;
- внешнюю квитанцию либо аудированное автоматическое/ручное решение.

Action version 1 требует policy `diagnostics:read`, имеет outcome `FOUND`, не требует idempotency key и принимает payload `{"identifier":"<value>"}`. Поддерживаются `correlationId`, `requestId`, `operationId`, `processId`, `stepInstanceId`, `jobId`, `executionId`, `attemptId`, `externalRequestId`, `messageId` и `decisionId`. Неизвестный identifier возвращает `404 diagnostics.trace_not_found`. Точная форма payload/result закреплена JSON Schemas `diagnostics-trace.payload.schema.json` и `diagnostics-trace.result.schema.json`.

## Поиск зависших операций

Реализуйте action `diagnostics.stalled`, возвращающий операции с затянувшимся ожиданием квитанции после исчерпания автоматических попыток доставки.

Action version 1 публикуется как `POST /api/diagnostics/stalled`, требует policy `diagnostics:read`, idempotency mode/scope `none` / `none`, включён и является default. Payload — `{}` без дополнительных полей. Ответ: HTTP 200, стандартный envelope с outcome `FOUND` и result `{"items":[...]}`; пустая выборка — `{"items":[]}`. Каждый элемент содержит только UUID `operationId`, UUID `processId` и непустой строковый `externalRequestId`. Операция встречается не более одного раза, сортировка — по `operationId` в канонической UUID-записи, пагинации нет.

Payload не по схеме возвращает `422 payload.invalid`, отсутствующий/невалидный JWT — HTTP 401, недостаточная policy — HTTP 403. Результат вычисляет зарегистрированная PostgreSQL-функция через generic runtime по текущим сохранённым данным. Action не меняет предметное состояние и не запускает доставку; access logs и action audit допустимы. Точная форма закреплена `diagnostics-stalled.payload.schema.json` и `diagnostics-stalled.result.schema.json`.

## Failpoint-профиль

Закрытый профиль реализует контракт из `07-autocheck-outline.md`: `COURSE_TEST_PROFILE=1`, одна точка в `COURSE_FAILPOINT` и JSON acknowledgement `failpoint.reached` перед блокировкой компонента. Обязательные имена: `after_job_claim`, `after_action_before_finish`, `after_outbox_claim`, `after_provider_response`, `after_inbox_saved`, `after_manual_decision`.

`after_action_before_finish` и `after_manual_decision` достигаются внутри ещё не завершённой транзакции; принудительная остановка должна привести к rollback. `after_inbox_saved` достигается после durable Inbox commit до применения workflow signal. Остальные точки находятся на границах, указанных в таблице failpoints автопроверки.

Потеря ответа после фактического принятия provider включается режимом `lost-response` выданного simulator. Production profile не активирует failpoints. Автопроверка дожидается acknowledgement, останавливает контейнер, удаляет failpoint и продолжает сценарий без гонки по времени.

## Практическое задание «Переживи сбой и докажи результат»

Реализуйте:

- конкурентный Outbox dispatcher;
- конкурентный Inbox reconciler;
- retry/backoff/jitter/dead-letter policy;
- recovery просроченных jobs и deliveries;
- обработку unknown outcome внешнего запроса;
- liveness/readiness API и worker;
- обязательные метрики;
- `diagnostics.trace` и `diagnostics.stalled`;
- безопасные structured logs;
- закрытый failpoint profile;
- автоматические аварийные tests;
- одну команду чистого запуска;
- одну команду полной проверки;
- итоговый README и troubleshooting guide.

### README и runbook

README содержит требования к окружению, входную конфигурацию, команды первого запуска и полной проверки, признаки readiness и безопасную диагностику типовых отказов.

В runbook приведите команды вызова `diagnostics.trace` и `diagnostics.stalled` и чтения их результатов.

## Открытая проверка

Публикационный пакет `moduledev-week-4-reliability-task` содержит Python checker, JSON Schemas, score manifest и фиксированные public fixtures.

```bash
./check.sh --repo /path/to/participant-solution
```

Checker выполняет cold build, regression сценарии недели 3, запускает два dispatcher/reconciler, проверяет outage/recovery provider, full restart, health, OpenMetrics, `diagnostics.trace`, регистрацию, доступ и формат `diagnostics.stalled`, затем пишет `week-4-public-report.json` без баллов и секретов. Детерминированные остановки по всем failpoints остаются скрытой проверкой, но их полный контракт опубликован заранее.

## Критерии приёмки

- Недоступность provider не теряет внешний запрос.
- Восстановление provider до исчерпания retries продолжает доставку автоматически.
- Исчерпание Outbox retries даёт `DEAD`, но оставляет процесс `WAITING_SIGNAL` с диагностическим признаком.
- Поздняя валидная квитанция продолжает такой процесс.
- Потеря ответа не создаёт второй внешний платёж.
- Два dispatcher не создают второй предметный эффект.
- Падение worker в каждой согласованной точке сохраняет инварианты.
- Падение adapter, API или reconciler до/после Inbox commit не теряет callback.
- Полный рестарт не нарушает идемпотентность повторной доставки.
- PostgreSQL сохраняет состояние при пересоздании Compose project за счёт объявленного named volume.
- `ready` и `live` имеют разную корректную семантику.
- Метрики отражают фактическое состояние очередей.
- Trace связывает действие, операцию, версию процесса, шаги, попытки и свидетельство результата.
- `diagnostics.stalled` возвращает выборку операций в опубликованном формате без изменения предметных данных.
- `payment-processing` не имеет предметного успеха без операции и валидной квитанции.
- `payment-review` не имеет успеха без аудированного решения.
- История событий и попыток остаётся append-only.
- В коде, image layers, конфигурации и логах нет секретов.

## Итоговый сценарий

Полная проверка на чистом окружении:

1. Собирает images и запускает Compose.
2. Публикует скрытый action и скрытую workflow-карту.
3. Создаёт и отправляет платёжную операцию.
4. Останавливает worker после claim и проверяет recovery.
5. Имитирует потерю ответа provider и повтор Outbox.
6. Доставляет раннюю и повторную квитанцию.
7. Параллельно выполняет ручные решения второго процесса.
8. Полностью перезапускает контур.
9. Проверяет SQL/API-проекции, метрики, trace и отсутствие дублей.
10. Формирует машинно читаемый отчёт.

## Скрытые проверки

Скрытый контур выбирает собственные точки остановки, число worker/dispatcher, задержки и порядок повторов. Он проверяет инварианты через API и стабильные read-only SQL views, а не имена классов или внутреннюю структуру решения.

## Оценка недели

Учебный зачёт `learning-outcome.1` требует подтверждённых условий результата недели 3
и сохранения результата при конкурентных исполнителях, stale completion и аварийных
границах commit недели 4. Полное техническое соответствие и баллы считаются отдельно.
Частные отклонения форматов остаются замечаниями; критическое непроверенное свойство
даёт «Недостаточно данных», доказанное нарушение обязательного условия — «Незачёт».

Помимо канонического вклада недели действует [внутренняя калибровка готовности сдачи](review-calibration-weeks-3-4.md).
На финальной неделе повышен вес самостоятельного чистого запуска, README/runbook и
одной команды полной проверки. Подтверждённый повтор ранее доставленного блокирующего
замечания усиливает ограничение диагностики; ошибки проверяющего повтором не считаются.

Неделя добавляет 30 баллов:

| ID | Баллы | Инвариант |
|---|---:|---|
| `PG-04` | 5 | Request/job/signal/manual repeat не создаёт второй предметный эффект |
| `REL-01` | 3 | Конкурентный Outbox dispatcher применяет timeout, retry и dead-letter policy |
| `REL-02` | 3 | Jobs и deliveries восстанавливаются после expiry/restart |
| `REL-03` | 3 | Provider outage/lost response не теряют и не удваивают payment |
| `REL-04` | 3 | API, worker, dispatcher и PostgreSQL-клиенты переживают full restart |
| `REL-05` | 3 | Детерминированные failpoints подтверждают commit-boundary invariants |
| `OBS-01` | 2 | Clean build/start выполняется одной командой |
| `OBS-02` | 2 | Открытые tests воспроизводимы |
| `OBS-03` | 2 | Health и metrics отражают состояние |
| `OBS-04` | 2 | Trace и logs связывают полный путь |
| `OBS-05` | 2 | В коде, конфигурации, images и логах нет secrets/лишних данных |
| **Итого** | **30** | |

## Стоп-факторы

К сдаче недели и итоговому рейтингу применяется единый перечень из `06-rating-and-checks.md`. Этот документ не переопределяет его формулировки. Обнаруженный фактор попадает в отчёт с точным названием из оценочного manifest.

## Артефакты недели

- итоговый контейнерный стенд;
- надёжные Python Outbox dispatcher и Inbox reconciler;
- health endpoints и OpenMetrics;
- actions `diagnostics.trace` и `diagnostics.stalled`;
- failpoint profile;
- полный открытый тестовый набор;
- машинно читаемый отчёт проверки;
- итоговый README и troubleshooting guide.

## Не входит в неделю

- обещание exactly-once;
- distributed transactions;
- Kubernetes deployment;
- production secret manager;
- ручная защита проекта как часть оценки;
- оценка промптов или истории AI-инструментов.
