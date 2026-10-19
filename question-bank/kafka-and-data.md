# Kafka и данные

## P0

1. Какую роль играют broker, topic, partition, record, offset и consumer group?
2. Как consumer group распределяет partitions между consumers и что такое rebalance?
3. Как выбрать partition key? Как он влияет на порядок и распределение нагрузки?
4. Как работают leader/follower и репликация partitions?
5. Чем отличаются at-most-once, at-least-once и exactly-once?
6. Когда коммитить offset для at-least-once и почему возможны дубликаты?
7. Для чего нужны retry и Dead Letter Queue?
8. Чем реляционная БД отличается от основных типов NoSQL-хранилищ?
9. Что такое индекс, нормализация и денормализация?
10. Что такое транзакция и какие свойства описывает ACID?
11. Чем партиционирование отличается от шардирования и какие проблемы они создают?

## P1

1. Что такое consumer lag, почему он растёт и как его наблюдать?
2. Что такое CAP и eventual consistency? В каких сценариях ослабленная согласованность приемлема?
3. Чем вертикальное масштабирование отличается от горизонтального?
4. Как репликация влияет на доступность, согласованность и задержку?
5. Как обрабатывать события, пришедшие не по порядку?
6. Почему Kafka не является обычной заменой PostgreSQL?

## P2

1. Для чего предназначены ClickHouse и Cassandra и чем их модели отличаются от реляционной?
2. Как Kafka Engine и materialized view используются для загрузки данных в ClickHouse?
3. Как partition key, clustering key, replication factor и consistency level работают в Cassandra?

