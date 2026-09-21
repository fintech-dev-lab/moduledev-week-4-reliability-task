# Контракт public checker недели 4

## Доверенные seams

- Compose services из задания;
- gateway на host-порту 8080 в исходном contract и на изолированном случайном loopback-порту при проверке;
- generic HTTP actions;
- provider v0.2.0 audit;
- PostgreSQL database `course`, host `psql`, login role `autocheck_reader` и read-only schema `autocheck`;
- fixed integration functions/roles;
- internal health/OpenMetrics endpoints;
- `diagnostics.trace` version 1;
- public fixtures с закреплённым digest.

Checker не вызывает `course.sh` на host и не использует physical tables/ORM/class names. Isolated override временно публикует PostgreSQL, provider и служебные HTTP endpoints на случайные loopback-порты: probes не выполняются внутри participant-controlled containers, а SQL session принудительно read-only.

Запуск из отдельного клона checker:

```bash
./check.sh --repo /path/to/participant-solution
```

`cli` обязан быть объявлен, но может успешно завершаться как one-shot service. Checker не требует имени `dotnet` у C# process: technology boundary подтверждается контрактом предыдущих недель, поведением services и неизменностью C# images в hidden run.

## Public phases

| Phase | Проверка |
|---|---|
| `admission` | Safe Compose, required replicas, Python runtimes и один locally built Python image |
| `startup` | Cold build, readiness, typed views, functions и roles |
| `outbox` | Provider outage, persisted retry, two-dispatcher resume, one provider payment |
| `receipt` | Missing/invalid signature, exact legacy mapping, duplicate/conflict |
| `review` | Limit-rule и manual branches |
| `observability` | Health, required OpenMetrics series, bounded labels и trace |
| `recovery` | Compose down/up без удаления named PostgreSQL volume, persisted state |
| `security` | Заявленные DB principals, отсутствие наблюдаемых runtime mismatches, узкая роль `autocheck_reader`, secret/full-message redaction |

## Hidden extensions

Hidden checker выполняет все шесть failpoints, `DEAD`/late receipt, expired delivery fencing, lost response, mixed-state restart и случайные interleavings. Он использует другие identifiers и интервалы, но тот же опубликованный контракт.

## Report

`week-4-public-report.json` содержит только status, named checks, safe expected/actual summaries и выполненные commands. Public artifact не содержит score, criterion IDs, payloads, secrets и exact hidden inputs.
