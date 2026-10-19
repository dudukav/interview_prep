# Kafka

- [Краткая справка](overview.md) — модель хранения и обработки событий.
- [Вопросы и ответы](theory.md) — самопроверка и подготовка к собеседованию.

Здесь под «базой» понимается устройство Kafka и её модель хранения; Kafka не заменяет обычную базу данных общего назначения.

## Подразделы

1. `01-core/` — broker, topic, partition, record, offset, retention, segment.
2. `02-producers/` — key, partitioning, batching, acknowledgements, idempotent producer.
3. `03-consumers/` — consumer group, rebalance, offset commit, consumer lag.
4. `04-delivery-and-failures/` — at-most/at-least/exactly-once, retries, DLQ, ordering.
5. `05-operations/` — replication, leader/follower, ISR, monitoring и изменение числа partitions.
6. `06-go-client/` — конфигурация клиента, сериализация, graceful shutdown, обработка ошибок.
7. `07-practice/` — локальный кластер и прикладные эксперименты.

## Практический маршрут

- поднять Kafka локально через Docker Compose;
- создать topic с несколькими partitions;
- написать Go producer и consumer;
- сравнить сообщения с ключом и без ключа;
- запустить несколько consumers в одной и разных группах;
- вручную управлять commit offset и воспроизвести повторную обработку;
- измерить consumer lag;
- добавить retry policy и DLQ;
- корректно остановить приложение без потери уже принятых сообщений.

## Вопросы первого приоритета

- Как связаны topic, partition, offset и consumer group?
- Где Kafka гарантирует порядок сообщений?
- Как выбрать partition key и что случится при плохом распределении ключей?
- Когда коммитить offset для at-least-once?
- Почему «exactly once» не означает автоматически ровно один бизнес-эффект во внешней БД?
- Что происходит при rebalance?
- Для чего нужны replication factor, leader/follower и ISR?
