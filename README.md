# Неделя 4. Надёжность и восстановление

Выдача — 24 сентября 2026 года. Дедлайн — **30 сентября, 23:59 МСК**.

Это финальное продолжение заданий недель 1–3 в том же репозитории участника. C# action/workflow runtime, PostgreSQL и Python-периметр сохраняют прежние границы. Участник доводит конкурентную доставку, recovery и наблюдаемость до проверяемого результата.

## Материалы

- [Условие задания](task/assignment.md)
- [Памятка участника](task/student-handout.md)
- [Контракт проверки](autocheck/README.md)
- [Действующая редакция полного контракта](docs/README.md)
- [Контракт наблюдаемости](docs/observability-contracts.md)
- [Внешние HTTP-контракты](docs/external-contracts.md)
- [Конфигурация и запуск checker](docs/configuration.md)
- [Рейтинг и стоп-факторы](docs/06-rating-and-checks.md)
- [Machine-readable contracts](contracts/course-1)
- [Diagnostics payload/result schemas](contracts/course-1)

## Проверяемый результат

Два dispatcher и два reconciler безопасно делят работу. Provider outage, lost response, lease expiry и полный restart не теряют запросы и не создают второй предметный эффект. Health, OpenMetrics и `diagnostics.trace` показывают путь операции по сохранённым фактам.

`diagnostics.stalled` возвращает операции с затянувшимся ожиданием квитанции после исчерпания автоматических попыток доставки.

## Открытая проверка

Требуются Python 3.11+, PostgreSQL client `psql`, Docker Engine и Docker Compose v2 с `!override`, `!reset` и `config --no-env-resolution`.

```bash
./check.sh --repo /path/to/participant-solution
```

Репозиторий checker не копируется в решение и не заменяет `check.sh` предыдущих недель.

Checker:

- выполняет Compose admission и cold build;
- поднимает отдельный project с synthetic secrets;
- выполняет regression сценарии недели 3;
- проверяет два dispatcher/reconciler из одного Python image;
- останавливает provider и проверяет durable retry с автоматическим продолжением;
- проверяет health/OpenMetrics и отсутствие high-cardinality labels;
- пересоздаёт весь Compose project без удаления PostgreSQL volume;
- проверяет `diagnostics.trace` по нескольким identifiers;
- проверяет регистрацию, доступ, формат и отсутствие предметных изменений у `diagnostics.stalled`;
- записывает `week-4-public-report.json` без баллов и секретов;
- удаляет project, volumes и созданные локальные images, если не передан `--keep-stack`.

Коды завершения:

- `0` — все public checks пройдены;
- `1` — нарушен контракт решения;
- `2` — checker или локальное окружение не готовы.

Checker не запускает host scripts сдачи и не читает физические предметные таблицы. HTTP/provider probes выполняются checker с временных случайных loopback-портов, а SQL evidence читается host `psql` под `autocheck_reader` в read-only session. Authoritative seams: HTTP actions, service health/OpenMetrics, provider audit и read-only schema `autocheck`.
