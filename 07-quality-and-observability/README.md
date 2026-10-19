# Качество и наблюдаемость

- [Краткая справка](overview.md) — стратегия тестирования и эксплуатационные сигналы.
- [Вопросы и ответы](theory.md) — самопроверка и подготовка к собеседованию.

## Подразделы

1. `01-testing/` — unit, integration, contract, E2E, test pyramid, Testcontainers.
2. `02-logging/` — уровни, structured logging, request/trace ID, персональные данные.
3. `03-metrics/` — RED/USE, Prometheus, Grafana, SLI/SLO, alerting.
4. `04-profiling/` — benchmarks, pprof, race detector, нагрузочное тестирование.
5. `05-ci-cd/` — format, lint, build, tests, security checks, image build, deploy.

## Практика

- добавить структурированные логи и провести один запрос через несколько слоёв;
- экспортировать latency, request rate и error rate;
- создать дашборд и простое оповещение;
- написать unit- и integration-тест для одного сценария;
- проверить сервис под нагрузкой и сопоставить ошибки с метриками;
- собрать минимальный CI pipeline, падающий при ошибке сборки или тестов.
