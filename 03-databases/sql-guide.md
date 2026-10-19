# SQL и реляционные БД: практическая основа

## 1. Как устроена реляционная схема

Возьмём упрощённую модель магазина:

```sql
CREATE TABLE users (
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email       TEXT NOT NULL UNIQUE,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE orders (
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id     BIGINT NOT NULL REFERENCES users(id),
    status      TEXT NOT NULL CHECK (status IN ('new', 'paid', 'cancelled')),
    total       NUMERIC(12, 2) NOT NULL CHECK (total >= 0),
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX orders_user_created_idx
    ON orders (user_id, created_at DESC);
```

Здесь таблицы выражают сущности, foreign key — связь «много заказов принадлежат одному пользователю», а ограничения защищают инварианты. Денежные значения хранятся в `NUMERIC` или целых минимальных единицах, а не в `float`.

## 2. Основные группы SQL-команд

- **DDL:** `CREATE`, `ALTER`, `DROP` — структура объектов.
- **DML:** `SELECT`, `INSERT`, `UPDATE`, `DELETE` — работа с данными.
- **TCL:** `BEGIN`, `COMMIT`, `ROLLBACK`, `SAVEPOINT` — транзакции.
- **DCL:** `GRANT`, `REVOKE` — права доступа.

На практике миграции должны быть версионированы, проверяемы и совместимы с поэтапным выпуском приложения.

## 3. Структура SELECT

```sql
SELECT u.id, u.email, count(o.id) AS paid_orders
FROM users AS u
LEFT JOIN orders AS o
    ON o.user_id = u.id
   AND o.status = 'paid'
WHERE u.created_at >= DATE '2026-01-01'
GROUP BY u.id, u.email
HAVING count(o.id) >= 2
ORDER BY paid_orders DESC, u.id
LIMIT 20;
```

Текст пишется начиная с `SELECT`, но логически запрос обрабатывается в другом порядке.

## 4. Логический порядок выполнения SELECT

Упрощённая последовательность:

```text
1. WITH
2. FROM
3. JOIN ... ON
4. WHERE
5. GROUP BY
6. агрегатные функции
7. HAVING
8. SELECT
9. DISTINCT
10. UNION / INTERSECT / EXCEPT
11. ORDER BY
12. LIMIT / OFFSET
```

Это объясняет несколько частых ситуаций:

- alias из `SELECT` обычно нельзя использовать в `WHERE`, потому что `WHERE` логически выполняется раньше;
- `WHERE` фильтрует строки до группировки, `HAVING` — уже сформированные группы;
- без `ORDER BY` порядок строк не гарантирован;
- `LIMIT` без устойчивого `ORDER BY` возвращает произвольный набор из подходящих строк.

Физический план может выполнять операции иначе, если результат остаётся эквивалентным.

## 5. Фильтрация и NULL

```sql
SELECT *
FROM orders
WHERE status = 'paid'
  AND created_at >= $1;
```

В приложении значения передают параметрами, а не склеивают со строкой SQL. Это предотвращает SQL injection и помогает переиспользовать планы.

`NULL` означает неизвестное или отсутствующее значение:

```sql
-- правильно
WHERE cancelled_at IS NULL

-- неверно: результат сравнения не TRUE, а UNKNOWN
WHERE cancelled_at = NULL
```

Выражение `NOT IN` с `NULL` в наборе часто даёт неожиданный результат. Для антисоединения обычно безопаснее `NOT EXISTS`:

```sql
SELECT u.*
FROM users AS u
WHERE NOT EXISTS (
    SELECT 1
    FROM orders AS o
    WHERE o.user_id = u.id
);
```

## 6. JOIN

`INNER JOIN` оставляет только совпавшие строки. `LEFT JOIN` сохраняет все строки слева.

Частая ошибка:

```sql
-- пользователи без заказов исчезнут: WHERE отфильтрует NULL
SELECT u.id, o.id
FROM users AS u
LEFT JOIN orders AS o ON o.user_id = u.id
WHERE o.status = 'paid';
```

Если нужны все пользователи и только оплаченные заказы, условие должно быть в `ON`:

```sql
SELECT u.id, o.id
FROM users AS u
LEFT JOIN orders AS o
  ON o.user_id = u.id
 AND o.status = 'paid';
```

JOIN может умножить строки. Если у пользователя три заказа, строка пользователя встретится три раза. `DISTINCT` не должен автоматически маскировать неправильную кардинальность — сначала нужно понять связи.

## 7. Агрегация

```sql
SELECT user_id,
       count(*) AS order_count,
       sum(total) AS total_spent
FROM orders
WHERE status = 'paid'
GROUP BY user_id
HAVING sum(total) >= 10000;
```

`COUNT(*)` считает строки. `COUNT(column)` считает только строки, где column не `NULL`. Все выбранные неагрегированные поля должны соответствовать группировке.

Условие по обычной колонке лучше помещать в `WHERE`, чтобы отсеять строки раньше. `HAVING` применяют для условий по результату агрегации.

## 8. Оконные функции

Оконная функция вычисляет значение по связанным строкам, не сворачивая их в одну строку:

```sql
SELECT id,
       user_id,
       total,
       row_number() OVER (
           PARTITION BY user_id
           ORDER BY created_at DESC, id DESC
       ) AS position
FROM orders;
```

Так решают задачи «последний заказ каждого пользователя», ranking, running total и сравнение строки с предыдущей. `PARTITION BY` формирует независимые группы окна, `ORDER BY` задаёт порядок внутри них.

## 9. CTE и подзапросы

CTE даёт имя промежуточному результату:

```sql
WITH paid AS (
    SELECT user_id, sum(total) AS total
    FROM orders
    WHERE status = 'paid'
    GROUP BY user_id
)
SELECT u.email, paid.total
FROM paid
JOIN users AS u ON u.id = paid.user_id;
```

CTE полезен для читаемости, рекурсии и разбиения сложной логики. Он не обязан быть быстрее подзапроса. Производительность проверяют по плану, а не по внешнему виду SQL.

## 10. INSERT, UPDATE и DELETE

PostgreSQL умеет возвращать изменённые строки:

```sql
INSERT INTO users (email)
VALUES ($1)
RETURNING id, created_at;
```

Атомарное изменение безопаснее последовательности read-modify-write:

```sql
UPDATE accounts
SET balance = balance - $1
WHERE id = $2
  AND balance >= $1
RETURNING balance;
```

Если запрос не вернул строку, средств недостаточно или account отсутствует. Две конкурирующие операции не смогут обе списать один и тот же остаток на основе устаревшего чтения.

Перед `UPDATE` или `DELETE` всегда проверяют условие. В production полезно сначала выполнить соответствующий `SELECT` в безопасной сессии или транзакции.

## 11. UPSERT

```sql
INSERT INTO processed_events (event_id, processed_at)
VALUES ($1, now())
ON CONFLICT (event_id) DO NOTHING;
```

`ON CONFLICT` опирается на unique constraint или unique index. Он помогает атомарно обрабатывать конкурирующие вставки и строить идемпотентность. Предварительный `SELECT`, затем `INSERT` без ограничения имеет race condition.

## 12. Транзакции

```sql
BEGIN;

UPDATE accounts
SET balance = balance - 100
WHERE id = 1;

UPDATE accounts
SET balance = balance + 100
WHERE id = 2;

COMMIT;
```

При ошибке выполняют `ROLLBACK`. В приложении важно гарантировать rollback на любом раннем выходе.

Транзакция должна включать только операции одной согласованной бизнес-единицы и быть короткой. Нельзя держать её открытой, пока пользователь отвечает или внешний HTTP-сервис выполняет запрос.

Для защиты конкурентной логики используют:

- один атомарный `UPDATE`;
- unique/check constraint;
- `SELECT ... FOR UPDATE`;
- optimistic locking через version;
- более строгий уровень изоляции и retry.

## 13. Как работает B-tree индекс

B-tree хранит отсортированные ключи в сбалансированном дереве. Верхние страницы направляют поиск к листовым, где находятся ключи и ссылки на строки. Высота дерева растёт медленно, поэтому точечный поиск требует небольшого числа переходов.

Индекс:

```sql
CREATE INDEX orders_user_created_idx
ON orders (user_id, created_at DESC);
```

подходит для:

```sql
WHERE user_id = $1
ORDER BY created_at DESC
LIMIT 20
```

и для диапазона `created_at` после равенства по `user_id`. Для запроса только по `created_at` ведущая колонка отсутствует, поэтому этот индекс обычно не оптимален.

## 14. Составные и специальные индексы

Порядок колонок выбирают по условиям равенства, диапазона и сортировки. Частое правило для B-tree: сначала колонки с равенством, затем диапазон или порядок. Но окончательное решение зависит от запросов и распределения значений.

Partial index хранит только нужную часть таблицы:

```sql
CREATE INDEX orders_open_idx
ON orders (created_at)
WHERE status = 'new';
```

Expression index помогает запросу по выражению:

```sql
CREATE UNIQUE INDEX users_email_lower_uidx
ON users (lower(email));
```

Тогда запрос должен использовать совместимое выражение `lower(email)`.

## 15. Почему индекс не всегда используется

Основные причины:

- запрос возвращает значительную часть таблицы;
- таблица маленькая;
- неподходящий порядок колонок;
- над индексированной колонкой применена несовместимая функция;
- тип параметра требует преобразования;
- статистика устарела или распределение сильно перекошено;
- условие недостаточно селективно;
- чтение heap-страниц дороже последовательного прохода.

Не нужно «заставлять» БД использовать индекс только потому, что он существует. Нужно сравнить реальные планы и время.

## 16. EXPLAIN и EXPLAIN ANALYZE

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM orders
WHERE user_id = 42
ORDER BY created_at DESC
LIMIT 20;
```

Смотреть следует на:

- типы узлов: Seq Scan, Index Scan, Join, Sort, Aggregate;
- estimated rows и actual rows;
- loops — сколько раз узел запускался;
- actual time;
- buffers — сколько страниц было прочитано;
- дорогие сортировки и временные данные;
- фильтры, удалившие много строк после чтения.

Большое расхождение estimated и actual rows часто указывает на проблему статистики или зависимые колонки. `EXPLAIN ANALYZE` действительно выполняет запрос, поэтому с `UPDATE`, `DELETE` и `INSERT` его нельзя бездумно запускать на production.

## 17. N+1 запрос

Проблема возникает, когда приложение делает один запрос списка, а затем отдельный запрос для каждой строки:

```text
1 запрос пользователей + N запросов заказов
```

Исправления:

- один JOIN, если результат не создаёт неудобного размножения;
- второй запрос `WHERE user_id = ANY($1)` и группировка в приложении;
- batch loader;
- заранее подготовленная read model.

Важно измерять число запросов, а не только длительность каждого по отдельности.

## 18. Пагинация

Offset pagination проста:

```sql
ORDER BY created_at DESC, id DESC
LIMIT 20 OFFSET 10000;
```

На больших offset БД всё равно должна пройти пропущенные строки, а при конкурентных вставках возможны пропуски и повторы.

Keyset pagination использует последний увиденный ключ:

```sql
WHERE (created_at, id) < ($1, $2)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

Она быстрее для последовательного просмотра и стабильнее при вставках. Порядок должен быть детерминированным, поэтому к неуникальному времени добавлен `id`.

## 19. Частые ошибки

### `SELECT *`

Читает ненужные данные, делает контракт неявным и может помешать index-only scan. Лучше перечислять нужные колонки.

### Нет `ORDER BY`

Физический порядок не гарантирован и может измениться после нового плана или обслуживания таблицы.

### Склеивание SQL-строки

Создаёт SQL injection. Значения передаются параметрами; динамические имена колонок выбираются только из разрешённого списка.

### Индекс на каждую колонку

Увеличивает размер, стоимость записи и обслуживания, но не обязательно помогает составным условиям.

### Слишком длинная транзакция

Удерживает connection, locks и старые версии строк, увеличивает конфликты и мешает очистке.

### Проверка уникальности только в коде

Два запроса проходят `SELECT` одновременно. Нужен unique constraint и обработка конфликта.

### Функция над индексированной колонкой

```sql
WHERE DATE(created_at) = DATE '2026-08-21'
```

Часто лучше диапазон:

```sql
WHERE created_at >= TIMESTAMPTZ '2026-08-21 00:00:00+03'
  AND created_at <  TIMESTAMPTZ '2026-08-22 00:00:00+03'
```

### Неограниченный запрос

Публичный endpoint должен иметь лимит, timeout и предсказуемую пагинацию.

## 20. Порядок работы над запросом

1. Сформулировать требуемый результат и кардинальность.
2. Выписать таблицы, связи и условия.
3. Начать с корректного простого запроса.
4. Проверить `NULL`, дубли и граничные случаи.
5. Добавить детерминированный `ORDER BY` и лимит.
6. Выполнить на реалистичных данных.
7. Посмотреть `EXPLAIN (ANALYZE, BUFFERS)`.
8. Найти самый дорогой этап, а не оптимизировать наугад.
9. Изменить запрос, схему или индекс.
10. Повторно измерить и сохранить объяснение решения.

