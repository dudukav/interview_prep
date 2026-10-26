# Отказоустойчивость и высокие нагрузки

- [Краткая справка](overview.md) — проектирование отказов, управление нагрузкой, масштабирование и профилирование Go.
- [Вопросы и ответы](theory.md) — самопроверка и подготовка к системным вопросам.

## Порядок изучения

1. SLI/SLO, error budget, availability и failure domains.
2. Timeouts, retries, idempotency, circuit breaker, bulkhead и load shedding.
3. Capacity: throughput, concurrency, latency, очереди и backpressure.
4. Вертикальное/горизонтальное масштабирование, stateless, кэши, БД и очереди.
5. Нагрузочные тесты и поиск точки насыщения.
6. Профилирование Go: benchmarks, `pprof`, trace, block/mutex profiles и race detector.
7. Rollout, graceful degradation, disaster recovery и проверка восстановления.

## Практический минимум

- задать SLO и вычислить допустимый error budget;
- добавить deadline и bounded retry с backoff/jitter к тестовому клиенту;
- сделать endpoint идемпотентным по idempotency key;
- ограничить concurrency и проверить поведение при переполнении;
- провести ступенчатый load test до точки насыщения;
- снять CPU и heap profiles под воспроизводимой нагрузкой;
- оптимизировать одно подтверждённое узкое место и сравнить профили/метрики до и после;
- выполнить rolling deploy и проверить graceful shutdown.

## Главный принцип

Надёжность и производительность — измеримые свойства системы, а не набор модных компонентов. Сначала формулируют пользовательскую цель и профиль нагрузки, затем находят ограничение по данным и только после этого меняют архитектуру или код.
