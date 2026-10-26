# Архитектура backend-систем

- [Краткая справка](overview.md) — основные архитектурные решения и компромиссы.
- [Вопросы и ответы](theory.md) — самопроверка и подготовка к собеседованию.

Этот раздел вторичен по отношению к Go, сетям и БД, но даёт контекст для проектных вопросов на собеседовании.

## Подразделы

1. `01-api-design/` — REST, OpenAPI, идемпотентность, versioning, pagination, errors.
2. `02-rpc-and-grpc/` — RPC, Protobuf, unary/streaming, deadlines, HTTP/2.
3. `03-service-architecture/` — монолит, модульный монолит, микросервисы, границы сервисов.
4. `04-reliability/` — timeout, retry/backoff, circuit breaker, bulkhead, rate limit.
5. `05-data-patterns/` — Transactional Outbox, Saga, CQRS, Event Sourcing.
6. `06-caching/` — cache-aside/read-through/write-through, TTL, eviction, invalidation.
7. `07-security/` — authentication, authorization, TLS, secrets и базовые угрозы API.

Вопросы из учебных защит, наиболее полезные для этого раздела, собраны в [банке вопросов](../question-bank/architecture.md).

Практическое продолжение паттернов надёжности, capacity planning и профилирование собраны в разделе [«Отказоустойчивость и высокие нагрузки»](../08-reliability-and-highload/README.md).
