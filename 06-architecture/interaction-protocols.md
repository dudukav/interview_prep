# Протоколы и способы взаимодействия сервисов

## Сначала разберём термины

В разговорах об API словом «протокол» часто называют разные уровни:

- **HTTP, HTTP/2, TCP, TLS** — сетевые протоколы;
- **SOAP** — протокол обмена XML-сообщениями;
- **REST** — архитектурный стиль, обычно реализованный поверх HTTP;
- **RPC** — модель взаимодействия как удалённого вызова процедуры;
- **gRPC** — RPC-фреймворк с Protobuf-контрактами, обычно поверх HTTP/2;
- **GraphQL** — язык запросов, схема типов и runtime для выполнения запросов;
- **WebSocket** — протокол постоянного двустороннего обмена;
- **SSE** — поток событий от сервера к клиенту поверх HTTP;
- **webhook** — соглашение, при котором одна система вызывает HTTP endpoint другой при событии;
- **Kafka/AMQP-сообщения** — асинхронное взаимодействие через broker.

Выбор делается не по принципу «что быстрее», а по требованиям к контракту, клиентам, задержке, согласованности, streaming, независимости команд и эксплуатации.

## Общая классификация

### Синхронное request-response

Клиент отправляет запрос и ждёт результат в рамках одного вызова. Так обычно работают REST, SOAP, unary gRPC и GraphQL.

Плюсы:

- простой ментальный сценарий;
- немедленный результат;
- понятная обработка ошибок;
- удобно для чтения и команд, результат которых нужен сейчас.

Минусы:

- latency складывается по цепочке;
- доступность caller зависит от downstream;
- долгие цепочки создают каскадные отказы;
- после timeout результат операции может быть неизвестен.

### Асинхронное взаимодействие

Producer передаёт сообщение broker или принимает команду в обработку, не ожидая завершения всей бизнес-операции.

Плюсы:

- сглаживание кратких пиков;
- временная развязка producer и consumer;
- независимое масштабирование consumers;
- естественная доставка одного события нескольким подписчикам.

Минусы:

- eventual consistency;
- дубли, порядок, retries и poison messages;
- сложнее отследить полный путь операции;
- принятие сообщения ещё не означает завершение бизнес-действия.

### Потоковое взаимодействие

Данные поступают частями в рамках длительного логического обмена. Streaming бывает server-side, client-side и bidirectional. Его поддерживают gRPC streams, WebSocket, SSE и брокеры сообщений, но семантика у них разная.

## REST поверх HTTP

REST моделирует систему как ресурсы с устойчивыми идентификаторами и единообразным интерфейсом. На практике REST API обычно использует JSON поверх HTTP.

### Ресурсы и URL

URL лучше описывает сущность, а HTTP method — действие:

```text
GET    /orders/42
POST   /orders
PATCH  /orders/42
DELETE /orders/42
POST   /orders/42/cancellation
```

`POST /cancelOrder` технически работает, но хуже использует единообразную HTTP-семантику. Не каждую бизнес-команду нужно насильно превращать в CRUD: подресурс действия часто понятнее искусственного обновления поля.

### Методы

| Метод | Обычная семантика | Safe | Idempotent |
|---|---|---:|---:|
| `GET` | прочитать представление | да | да |
| `HEAD` | получить headers без body | да | да |
| `POST` | создать или выполнить команду | нет | нет по умолчанию |
| `PUT` | полностью заменить ресурс по известному URI | нет | да |
| `PATCH` | частично изменить | нет | зависит от формата/операции |
| `DELETE` | удалить ресурс | нет | да по итоговому состоянию |

Safe означает, что метод не должен изменять бизнес-состояние. Idempotent означает одинаковый итоговый эффект при повторе, а не обязательно одинаковый status code или отсутствие записей аудита.

### Коды ответа

- `200 OK` — успешный ответ с body;
- `201 Created` — ресурс создан, полезен header `Location`;
- `202 Accepted` — команда принята, но ещё не завершена;
- `204 No Content` — успех без body;
- `400 Bad Request` — запрос синтаксически или структурно неверен;
- `401 Unauthorized` — нет корректной аутентификации;
- `403 Forbidden` — субъект известен, но действие запрещено;
- `404 Not Found` — ресурс не найден или намеренно скрыт;
- `409 Conflict` — конфликт с текущим состоянием;
- `412 Precondition Failed` — не выполнено условие `If-Match` и т. п.;
- `422 Unprocessable Content` — структура понятна, но семантическая валидация не прошла;
- `429 Too Many Requests` — rate limit;
- `500` — неожиданная внутренняя ошибка;
- `502/503/504` — проблемы upstream, временная недоступность или timeout gateway.

Ошибка должна иметь машинный код, безопасное сообщение, детали допустимых полей и trace/request ID. Stack trace и SQL клиенту не возвращают.

### Идемпотентность и конкурентное изменение

Для небезопасного повтора `POST` клиент передаёт `Idempotency-Key`. Сервер связывает ключ с параметрами и результатом первой операции. Повтор с тем же ключом и другими параметрами должен быть отклонён.

Потерянное обновление предотвращают optimistic concurrency через version/ETag:

```text
GET /orders/42        -> ETag: "v7"
PATCH /orders/42
If-Match: "v7"       -> 200 или 412, если версия уже изменилась
```

### Пагинация, фильтрация и кэш

Offset pagination проста, но на меняющихся больших наборах может пропускать/дублировать элементы и становится дорогой. Cursor pagination использует стабильный ключ продолжения и лучше подходит для больших лент.

HTTP caching опирается на `Cache-Control`, `ETag`, `If-None-Match`, `Last-Modified` и `304 Not Modified`. Кэшировать нужно по явной политике, особенно для персональных данных и авторизованных ответов.

### Контракт и версионирование

OpenAPI описывает paths, методы, параметры, schemas и ответы. Contract-first помогает обсудить API и сгенерировать clients/stubs, но не гарантирует хорошую модель.

Совместимые изменения: новое необязательное поле, новый endpoint, новый необязательный query parameter. Несовместимые: удаление/переименование поля, изменение типа или смысла, новый обязательный input, сужение допустимых значений.

Версию можно отражать в path (`/v1`), header или media type. Важнее самого места правила совместимости, deprecation, срок миграции и наблюдение использования старой версии.

### Когда REST подходит

- публичный или browser-friendly API;
- CRUD и resource-oriented модель;
- важны стандартные HTTP intermediaries, caching и простая диагностика;
- клиенты написаны на разных языках и не хотят обязательной генерации кода.

## SOAP

SOAP — протокол XML-сообщений с формальной структурой Envelope/Header/Body. Он может работать поверх HTTP и других транспортов, хотя HTTP встречается чаще всего.

```xml
<soap:Envelope xmlns:soap="http://www.w3.org/2003/05/soap-envelope">
  <soap:Header>
    <auth:Token xmlns:auth="urn:example:auth">...</auth:Token>
  </soap:Header>
  <soap:Body>
    <m:GetOrder xmlns:m="urn:example:orders">
      <m:orderId>42</m:orderId>
    </m:GetOrder>
  </soap:Body>
</soap:Envelope>
```

### WSDL и XSD

WSDL описывает операции, сообщения, типы, binding и endpoint сервиса. XML Schema (XSD) задаёт типы и ограничения XML. По контракту инструменты генерируют клиентский и серверный код.

SOAP API обычно operation-oriented: `GetOrder`, `CreatePayment`, а не resource-oriented. HTTP method часто остаётся `POST`, поэтому прикладной результат выражается SOAP body, а не только status code.

### SOAP Fault

Стандартная ошибка передаётся как `Fault`: code, reason и application-specific details. HTTP `500` может сопровождать SOAP Fault даже для ожидаемой прикладной ошибки, поэтому клиент обязан разбирать SOAP-сообщение, а не судить только по HTTP.

### WS-* стандарты

Экосистема включает WS-Security, WS-Addressing, WS-ReliableMessaging и другие спецификации. WS-Security может подписывать и шифровать части сообщения на уровне XML, что отличается от TLS, защищающего транспортный канал.

Плюсы SOAP:

- строгий формальный контракт и развитая генерация;
- стандарты enterprise-интеграции;
- message-level security/signatures;
- привычен в банках, госинтеграциях и legacy enterprise.

Минусы:

- многословный XML и сложная экосистема;
- тяжёлая ручная отладка;
- namespace/XSD/WS-* добавляют порог входа;
- обычно хуже подходит для web/mobile API.

SOAP выбирают не потому, что он «старый, но надёжный», а когда этого требует существующий контракт, регуляторная интеграция или конкретные WS-* возможности.

## RPC и gRPC

RPC представляет взаимодействие как вызов метода удалённого сервиса:

```text
OrderService.GetOrder(GetOrderRequest) -> GetOrderResponse
```

Это удобная модель, но вызов не становится локальной функцией: остаются latency, частичный отказ, deadline, retry и неопределённый результат.

### Protobuf-контракт

```proto
syntax = "proto3";

package orders.v1;

service OrderService {
  rpc GetOrder(GetOrderRequest) returns (GetOrderResponse);
  rpc WatchOrders(WatchOrdersRequest) returns (stream OrderEvent);
}

message GetOrderRequest {
  int64 order_id = 1;
}

message GetOrderResponse {
  Order order = 1;
}
```

Protobuf задаёт типы и числовые field tags. Wire format бинарный и компактный; generated code уменьшает ручные ошибки сериализации.

### Виды gRPC-вызовов

- unary — один request, один response;
- server streaming — один request, поток responses;
- client streaming — поток requests, один response;
- bidirectional streaming — независимые потоки в обе стороны.

Streaming не означает автоматическую бесконечную масштабируемость. Нужны backpressure, ограничение размера сообщений, deadlines, keepalive и контроль медленных consumers.

### HTTP/2

gRPC обычно использует HTTP/2: binary framing, multiplexing streams в одном connection и flow control. Потеря TCP packet всё ещё может временно задержать streams одного connection. HTTP/2 не отменяет application-level limits.

### Metadata, deadlines и cancellation

Metadata переносит authentication, tracing и служебные значения. Deadline должен идти от caller по всей цепочке. При отмене context сервер прекращает ненужную работу кооперативно.

Клиент должен задавать deadline явно. Вызов без deadline способен ждать неограниченно долго в зависимости от библиотечных настроек.

### Status codes и details

gRPC имеет собственные статусы: `InvalidArgument`, `NotFound`, `AlreadyExists`, `FailedPrecondition`, `Unauthenticated`, `PermissionDenied`, `ResourceExhausted`, `Unavailable`, `DeadlineExceeded`, `Internal` и другие.

`Unavailable` может допускать retry, `InvalidArgument` — обычно нет. Повтор зависит также от идемпотентности и оставшегося deadline. Structured error details передают дополнительные типизированные данные; парсить текст сообщения нельзя.

### Эволюция Protobuf

- нельзя менять значение существующего field tag;
- удалённый tag и имя помечают `reserved`;
- новое поле обычно добавляется с новым tag;
- старый consumer проигнорирует неизвестное поле;
- отсутствие scalar field в proto3 требует внимания: для различения «не передано» и zero value используют `optional`, wrapper или отдельную модель;
- изменение семантики при сохранении типа может быть несовместимо логически.

### gRPC в браузере

Обычный browser JavaScript не всегда может напрямую использовать полный gRPC transport. Применяют gRPC-Web или gateway/transcoding в REST/JSON. Это добавляет инфраструктуру и может ограничивать streaming.

### Когда gRPC подходит

- внутреннее взаимодействие сервисов;
- строгая типизация и code generation;
- polyglot-команды с единым `.proto`;
- низкий overhead и streaming;
- контролируемые клиенты и инфраструктура HTTP/2.

Он хуже подходит, если API должны легко вызывать вручную, использовать произвольные browser clients или кэшировать стандартными HTTP/CDN-механизмами.

## GraphQL

GraphQL задаёт типизированную schema и позволяет клиенту указать форму нужного результата.

```graphql
query OrderPage($id: ID!) {
  order(id: $id) {
    id
    status
    customer { name }
    items { title quantity }
  }
}
```

Обычно есть:

- query для чтения;
- mutation для изменения;
- subscription для событий;
- schema с object, scalar, enum, interface/union и input types;
- resolvers, получающие поля.

Плюсы:

- клиент запрашивает только нужные поля;
- один schema объединяет несколько backend-возможностей;
- удобен для UI с разными представлениями;
- introspection и сильные инструменты разработки.

Риски:

- N+1 в resolvers, для чего нужны batching/DataLoader;
- дорогие вложенные запросы;
- сложность HTTP/CDN caching;
- авторизация должна проверяться на уровне операций и объектов/полей;
- одна HTTP `200` response может содержать `errors` вместе с частичным `data`;
- schema может превратиться в связанную «распределённую БД» без ясных владельцев.

Защита включает depth/complexity limits, persisted queries, timeout, rate/cost limits и observability по operation name. Запрещать introspection само по себе недостаточно.

GraphQL подходит для client-driven чтения и BFF/API gateway, но не обязан заменять внутренние REST/gRPC APIs.

## WebSocket

WebSocket начинается HTTP handshake, затем переходит к постоянному полнодуплексному соединению с frames. Обе стороны могут отправлять данные независимо.

Подходит для чатов, multiplayer, collaborative editing и интерактивных live-событий. Необходимо спроектировать:

- application message schema и versioning;
- authentication при handshake и повторную проверку authorization;
- ping/pong и idle timeout;
- bounded outgoing queue и политику медленного клиента;
- reconnect с backoff+jitter;
- sequence number и восстановление пропущенного состояния;
- распределение connection state между instances;
- graceful drain при deploy.

WebSocket — транспорт канала, а не готовая семантика событий, delivery guarantee или replay.

## Server-Sent Events

SSE отправляет события только от сервера к клиенту поверх обычного HTTP response с `text/event-stream`:

```text
id: 1842
event: order-status
data: {"order_id":42,"status":"paid"}

```

Browser `EventSource` автоматически переподключается и может передать `Last-Event-ID`. SSE проще WebSocket для notifications, прогресса и live feeds, когда сообщения от клиента идут обычными HTTP-запросами.

Ограничения: одно направление, text format, особенности proxy buffering/timeouts и лимит долгих connections. Сервер всё равно должен решить replay и поведение медленного клиента.

## Polling и long polling

Short polling регулярно спрашивает состояние. Он прост, хорошо работает через HTTP-инфраструктуру, но создаёт пустые запросы и задержку до следующего интервала.

Long polling удерживает запрос до события или timeout, затем клиент сразу открывает новый. Он уменьшает пустой трафик, но усложняет capacity planning и всё ещё создаёт churn запросов.

Для редких обновлений polling часто дешевле по сложности, чем постоянный канал.

## Webhooks

Webhook — исходящий HTTP callback при событии. Provider вызывает endpoint consumer. Поскольку сети и consumers ненадёжны, production webhook требует:

- уникальный event ID;
- подпись тела и timestamp, например HMAC;
- защиту от replay по допустимому окну времени;
- быстрый `2xx` после durable acceptance;
- retries с backoff и конечным сроком;
- at-least-once ожидание и идемпотентный consumer;
- порядок не должен предполагаться без явной гарантии;
- журнал попыток, ручной replay и ротацию secret;
- проверку URL и защиту от SSRF при регистрации endpoint.

Подпись считают по исходным bytes body до JSON parsing, иначе изменение пробелов/порядка полей нарушит проверку.

## Асинхронные сообщения и события

### Command, event и query

- query просит данные и не должен менять бизнес-состояние;
- command просит конкретного владельца выполнить действие и может быть отклонена;
- event сообщает факт, который уже произошёл, и не адресован одному конкретному получателю.

Назвать команду событием недостаточно. `CreateOrder` — команда, `OrderCreated` — факт в прошедшем времени.

### Delivery semantics

- at-most-once: возможна потеря, но нет broker-level redelivery;
- at-least-once: сообщение может прийти повторно, consumer обязан обрабатывать дубли;
- exactly-once обычно ограничено конкретным контуром/транзакцией и не гарантирует один внешний бизнес-эффект.

Ordering обычно гарантируется только внутри partition/key, а не глобально. Выбор partition key влияет на порядок и равномерность нагрузки.

### Schema evolution

Для JSON/Avro/Protobuf events задают schema, owner и compatibility policy. Consumer должен переносить неизвестные поля и отсутствие новых optional fields. Dual read/write или upcasters помогают при миграции, но бесконечная поддержка всех версий дорога.

### Transactional Outbox

Чтобы изменение БД и публикация события не расходились, приложение в одной локальной транзакции пишет бизнес-данные и outbox record. Отдельный publisher отправляет outbox в broker. Возможны дубли после отправки до отметки, поэтому consumer всё равно идемпотентен.

DLQ не является автоматическим лечением. Нужны причина помещения, alert, владелец, исправление и безопасный replay.

## Сравнение подходов

| Подход | Модель | Контракт | Сильные стороны | Основные риски | Типичные случаи |
|---|---|---|---|---|---|
| REST/HTTP | request-response, resources | OpenAPI/JSON Schema, может быть нестрогим | простота, браузеры, HTTP caching, отладка | разнобой контрактов, over/under-fetching | публичные API, CRUD |
| SOAP | operation/message | WSDL + XSD | строгий enterprise-контракт, WS-* | XML и эксплуатационная сложность | банки, гос/legacy интеграции |
| gRPC | RPC, unary/streaming | Protobuf | типизация, codegen, compact transport, streams | browser/proxy tooling, теснее связан с generated contract | внутренние сервисы |
| GraphQL | client-defined query | GraphQL schema | гибкость UI, единая graph schema | N+1, query cost, caching и authorization | BFF, сложные UI |
| WebSocket | full-duplex stream | прикладная schema | низкая задержка в обе стороны | stateful connections, reconnect/backpressure | чат, collaboration |
| SSE | server-to-client stream | прикладная event schema | просто для browser и однонаправленных updates | proxy buffering, только server -> client | notifications, progress |
| Webhook | event callback по HTTP | документация/OpenAPI + event schema | интеграция без постоянного polling | retries, подписи, replay, SSRF | внешние события партнёрам |
| Broker | async command/event | event schema/registry | decoupling, buffering, fan-out | eventual consistency, дубли, lag, DLQ | интеграция доменов, pipelines |

## Как выбирать

Задайте вопросы в таком порядке:

1. Нужен ли результат прямо сейчас? Если нет, рассмотрите broker/job и асинхронную модель.
2. Кто клиенты: браузеры, внешние партнёры или контролируемые сервисы?
3. Модель resource-oriented или operation-oriented?
4. Нужен ли строгий generated contract?
5. Есть ли server/client/bidirectional streaming?
6. Нужны ли HTTP caching, CDN и простая ручная диагностика?
7. Допустима ли eventual consistency и повторная доставка?
8. Как эволюционирует schema и кто владелец контракта?
9. Какие требования к security, audit и message signing?
10. Поддерживает ли команда инфраструктуру и observability выбранного подхода?

Практические ориентиры:

- public API по умолчанию — REST/HTTP, если нет иной причины;
- внутренний типизированный RPC с контролируемыми клиентами — gRPC;
- сложная выборка для нескольких UI — GraphQL/BFF;
- legacy enterprise contract — SOAP;
- двусторонний real-time — WebSocket;
- server-to-browser updates — SSE;
- уведомление внешней системы — webhook;
- независимая обработка, fan-out и сглаживание пиков — broker.

Система может разумно сочетать несколько способов: REST наружу, gRPC внутри синхронного контура, Kafka для domain events и SSE для обновления UI. Важно не плодить технологии без необходимости и иметь единые правила contracts, deadlines, security и observability.

## Общие требования независимо от технологии

### Надёжность

- deadline на каждый удалённый вызов;
- retry только для подходящей ошибки и идемпотентной операции;
- exponential backoff, jitter и retry budget;
- limits на размер сообщения, concurrency и очереди;
- circuit breaker/load shedding при перегрузке;
- graceful shutdown и drain;
- явная delivery и ordering semantics.

### Безопасность

- TLS/mTLS по модели угроз;
- authentication и authorization на доверенной стороне;
- минимальные scopes/permissions;
- валидация входа и ограничение размера/сложности;
- защита secrets и чувствительных данных;
- replay protection для подписанных сообщений;
- SSRF-защита исходящих callbacks;
- rate limiting и audit чувствительных операций.

### Наблюдаемость

- correlation/trace context через все hops;
- RED metrics по operation/method без high-cardinality IDs;
- latency и status каждого dependency;
- structured logs с безопасными полями;
- broker lag, oldest message age, redelivery и DLQ;
- количество активных streams/connections и dropped messages;
- версия клиента/контракта для миграций.

### Тестирование контракта

- schema validation и breaking-change checks в CI;
- generated client/server compilation;
- consumer/provider contract tests;
- integration tests с реальным protocol stack;
- негативные проверки: timeout, malformed input, duplicate, out-of-order, partial response;
- совместимость старого клиента с новым сервером и наоборот.

## Частые ошибки на собеседованиях и в проектах

- называть REST сетевым протоколом;
- считать gRPC всегда быстрее без профиля payload и инфраструктуры;
- думать, что Protobuf автоматически делает изменение совместимым;
- повторять любой `POST`/RPC после timeout;
- считать HTTP `200` гарантией отсутствия GraphQL errors;
- проектировать WebSocket без bounded queues и resynchronization;
- доверять webhook без подписи и защиты от replay;
- считать Kafka «exactly once» гарантией единственного платежа;
- использовать broker для сценария, где caller немедленно нужен результат, не спроектировав status API;
- смешивать transport status и бизнес-статус операции;
- выбирать технологию только по скорости сериализации, игнорируя поддержку и диагностику.
