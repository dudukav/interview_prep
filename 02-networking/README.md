# Сети и соединения

- [Краткая справка](overview.md) — как устроен сетевой обмен и что важно backend-разработчику.
- [Вопросы и ответы](theory.md) — самопроверка и подготовка к собеседованию.

## Иерархия

```text
Приложение:       HTTP/1.1, HTTP/2, WebSocket
Транспорт:        TCP, UDP
Сетевой уровень:  IP
Канальный:        Ethernet / Wi-Fi
```

HTTP обычно работает поверх TCP; HTTPS добавляет TLS между HTTP и TCP. WebSocket начинает работу как HTTP-запрос с переключением протокола и затем использует двусторонний канал поверх TCP. UDP не устанавливает соединение и не гарантирует доставку или порядок пакетов.

## Порядок изучения

1. Основы — IP, CIDR, routes, NAT, DNS, порты, sockets, latency/throughput.
2. TCP/UDP — handshake, поток и framing, retransmission, flow/congestion control, datagrams.
3. HTTP/TLS — запрос/ответ, методы, коды, заголовки, cookies, сертификаты, HTTP/1.1–3.
4. Инфраструктура — reverse proxy, load balancer, service discovery и connection pools.
5. Долгие соединения — WebSocket, heartbeat, backpressure и reconnect.
6. Диагностика — `curl`, `dig`, `ss`/`netstat`, `lsof`, `nc`, Wireshark/tcpdump, логи и метрики.

## Практический минимум

| Тема | Задача |
|---|---|
| TCP | Написать echo server/client, проверить несколько клиентов, таймауты и закрытие соединения |
| UDP | Отправить datagrams, искусственно показать потерю/перестановку, добавить прикладной request ID |
| HTTP | Поднять сервер, исследовать запрос через `curl -v`, настроить timeouts, graceful shutdown и TLS |
| WebSocket | Сделать чат или поток статусов, добавить ping/pong, дедлайн чтения и reconnect клиента |
| Диагностика | Найти процесс на порту, посмотреть открытые соединения и сопоставить сетевую ошибку с логами |
| Proxy/LB | Поставить reverse proxy перед двумя экземплярами, проверить forwarded headers и отказ одного backend |

Для каждого опыта сохранять: схему, команды запуска, ожидаемый результат, фрагмент логов и объяснение наблюдаемого поведения.
