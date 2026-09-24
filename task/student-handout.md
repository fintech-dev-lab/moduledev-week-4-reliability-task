# Неделя 4. Памятка участника

Нормативные детали: [полный контракт](../docs/05-week-4.md), [наблюдаемость](../docs/observability-contracts.md), [конфигурация и запуск checker](../docs/configuration.md).

## Перед изменениями

Учебный зачёт требует обязательного предметного результата и восстановления при
конкуренции и авариях. Отдельное отклонение формата не равно потере платежа или истории:
замечание сохраняется, а отказ в зачёте требует доказанного критического последствия.
Непроверенное критическое свойство означает «Недостаточно данных».

Финальная сдача должна воспроизводиться другим разработчиком по README/runbook:
первый запуск, конфигурация, readiness, полная проверка и безопасная диагностика.
Повторный ручной запуск и исправления проверяющего не заменяют готовую поставку.
До сдачи перепроверьте ранее доставленные замечания; подтверждённые повторы учитываются строже.

1. Зафиксируйте working end-to-end недели 3.
2. Составьте таблицу `граница -> durable fact -> owner recovery`.
3. Сначала добавьте вторые replicas и fencing, затем failpoints.
4. Не лечите race увеличением `sleep`.

До запуска checker добавьте `autocheck_reader`, declared PostgreSQL named volume и все переменные из configuration contract, включая job/worker test intervals.

## Идентичность retry

```text
jobId         stable for logical job
executionId   stable for subject effect
attemptId     new for each attempt
leaseVersion  increases on claim/reclaim

externalRequestId  stable for all provider attempts
outboxId           stable for one delivery
```

HTTP выполняйте вне claim transaction. Результат принимайте только условным update по owner/lease version.

## DEAD и late receipt

`DEAD` отвечает только на вопрос о delivery attempts. Provider мог принять request до потери response. Поэтому operation остаётся `PROCESSING`, process остаётся `WAITING_SIGNAL`, а поздняя валидная receipt имеет приоритет.

Реализуйте `diagnostics.stalled`: action возвращает операции с затянувшимся ожиданием квитанции после исчерпания автоматических попыток доставки. Version 1, policy `diagnostics:read`, payload `{}`, outcome `FOUND`, result `{"items":[...]}`. Элемент содержит `operationId`, `processId`, `externalRequestId`; action не меняет предметное состояние. Полный транспортный контракт — в `docs/observability-contracts.md`.

## Crash matrix

| Точка | Durable до сбоя | После restart |
|---|---|---|
| after job claim | lease | reclaim после expiry |
| after action before finish | ничего из transaction | rollback и retry |
| after Outbox claim | delivery lease | reclaim и stable request |
| after provider response | provider effect возможен | retry с тем же key |
| after Inbox saved | Inbox receipt | reconciler применяет signal |
| after manual decision | transaction не завершена | repeat создаёт одно decision |

## Наблюдаемость

- Metric count и oldest age сверяйте с persisted state.
- IDs храните в logs/trace, не в names или values metric labels.
- Trace собирайте по явным связям, не парсингом log text.
- Ready проверяет критические dependencies; live не является deep health check.
- Provider outage отражайте delivery state и metrics, а не readiness API/worker/dispatcher.

## Public checker

Держите репозиторий задания отдельно от решения:

```bash
./moduledev-week-4-reliability-task/check.sh --repo /path/to/participant-solution
```

Отчёт `week-4-public-report.json` является локальным диагностическим artifact и не входит в сдачу.

## Типовые ошибки

- один `OUTBOX_OWNER` у двух dispatcher;
- HTTP внутри transaction с заблокированной Outbox row;
- бесконечные retries или retry terminal 4xx;
- новый external key/body на retry;
- `DEAD` переводит operation в `REJECTED`;
- provider включён в readiness dependency;
- UUID используются как metric labels;
- trace строится из unstructured logs;
- failpoint активируется публичным HTTP endpoint;
- recovery требует ручного SQL.
