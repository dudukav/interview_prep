# Архитектура backend-систем

- [Краткая справка](overview.md) — основные архитектурные решения и компромиссы.
- [Протоколы взаимодействия](interaction-protocols.md) — REST, SOAP, gRPC, GraphQL, WebSocket, SSE, webhooks и сообщения через broker.
- [Вопросы и ответы](theory.md) — самопроверка и подготовка к собеседованию.

Этот раздел вторичен по отношению к Go, сетям и БД, но даёт контекст для проектных вопросов на собеседовании.

## Подразделы

1. [Протоколы взаимодействия](interaction-protocols.md) — выбор между REST, SOAP, gRPC, GraphQL, streaming и messaging.
2. API design — OpenAPI, идемпотентность, versioning, pagination и модель ошибок.
3. RPC/gRPC — Protobuf, unary/streaming, deadlines, HTTP/2 и совместимость.
4. Service architecture — монолит, модульный монолит, микросервисы, границы сервисов.
5. Reliability — timeout, retry/backoff, circuit breaker, bulkhead, rate limit.
6. Data patterns — Transactional Outbox, Saga, CQRS, Event Sourcing.
7. Caching — cache-aside/read-through/write-through, TTL, eviction, invalidation.
8. Security — authentication, authorization, TLS, secrets и базовые угрозы API.

Вопросы из учебных защит, наиболее полезные для этого раздела, собраны в [банке вопросов](../question-bank/architecture.md).

Практическое продолжение паттернов надёжности, capacity planning и профилирование собраны в разделе [«Отказоустойчивость и высокие нагрузки»](../08-reliability-and-highload/README.md).
