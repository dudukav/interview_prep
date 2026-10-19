# Задачи с подвохом

Это собственные задачи для подготовки, не перенесённые из исходного репозитория. Они не требуют алгоритмов. Для каждой задачи нужно:

1. Предсказать результат до запуска.
2. Объяснить его через правила языка.
3. Запустить обычную версию и при необходимости `go test -race`.
4. Выполнить указанное изменение и снова объяснить результат.

Ответы лучше хранить рядом в собственных файлах, не изменяя условие.

## Основы языка

### 1. Когда вычисляется defer?

```go
func main() {
    x := 1
    defer fmt.Println("A", x)
    defer func() { fmt.Println("B", x) }()
    x = 2
}
```

- Предскажите две строки и их порядок.
- Измените только один аргумент или вызов так, чтобы обе строки вывели `2`.
- Затем сделайте так, чтобы порядок строк стал обратным, не перемещая `x = 2`.

### 2. Затенённая ошибка

```go
func load(enabled bool) (err error) {
    if enabled {
        value, err := strconv.Atoi("bad")
        _ = value
        if err != nil {
            fmt.Println("inside:", err)
        }
    }
    return err
}
```

- Что возвращает `load(true)`?
- Измените одну строку так, чтобы возвращалась ошибка и `value` оставалась доступна только внутри `if`.
- Почему простая замена `:=` на `=` сразу не компилируется?

### 3. Named result и defer

```go
func answer() (n int) {
    defer func() { n++ }()
    return 41
}
```

- Что вернёт функция?
- Что изменится, если убрать имя `n` и вернуть `int`?
- Перепишите функцию без named result, сохранив результат, но не меняя literal `41`.

### 4. Pointer на копию

```go
type User struct{ Name string }

func rename(u User) {
    u.Name = "Bob"
}
```

- Почему исходный User не меняется?
- Измените только сигнатуру и одну строку вызова, чтобы изменение сохранилось.
- Затем сохраните value receiver, но верните изменённое значение без pointer.

## Slices, maps и strings

### 5. Общий backing array

```go
a := []int{1, 2, 3, 4}
b := a[:2]
b = append(b, 9)
fmt.Println(a)
fmt.Println(b)
```

- Предскажите результат.
- Измените создание `b` одной строкой так, чтобы append не изменил `a`, не копируя элементы вручную.
- Объясните роль полного slice expression `a[:2:2]`.

### 6. Копия slice не копирует элементы

```go
a := []int{1, 2, 3}
b := a
b[0] = 8
b = append(b, 4)
fmt.Println(a, b)
```

- Какие изменения видны через `a`?
- Как результат зависит от исходной capacity?
- Сделайте `b` полностью независимым с помощью `copy`, затем с помощью `slices.Clone`.

### 7. Nil map

```go
var counts map[string]int
fmt.Println(counts["go"])
counts["go"]++
```

- Что успеет выполниться?
- Почему чтение допустимо, а запись нет?
- Добавьте минимальное изменение, сохранив объявление через `var`.

### 8. Отсутствие или zero value?

```go
m := map[string]int{"ready": 0}
fmt.Println(m["ready"] == m["missing"])
```

- Что будет выведено?
- Измените код так, чтобы отдельно определить наличие каждого ключа.
- Спроектируйте функцию `Lookup`, которая не смешивает отсутствие и нулевое значение.

### 9. Bytes, runes и видимые символы

```go
s := "Go界"
fmt.Println(len(s))
fmt.Println(len([]rune(s)))
for i, r := range s {
    fmt.Println(i, r)
}
```

- Предскажите длины и значения индексов.
- Измените строку, добавив combining mark, и объясните, почему число runes может не совпасть с числом видимых символов.
- Напишите безопасное получение первых двух runes без разрезания UTF-8.

### 10. Удержание большого массива

```go
func prefix(data []byte) []byte {
    return data[:10]
}
```

- Почему результат может удерживать в памяти весь большой `data`?
- Измените функцию так, чтобы результат владел только необходимыми bytes.
- Сравните варианты через `copy` и `bytes.Clone`.

## Методы и интерфейсы

### 11. Method set

```go
type Closer interface{ Close() error }
type File struct{}

func (*File) Close() error { return nil }

var _ Closer = File{}
```

- Почему программа не компилируется?
- Измените одну лексему, чтобы compile-time проверка прошла.
- Что изменится, если receiver сделать value receiver?

### 12. Typed nil error

```go
type AppError struct{ Message string }
func (e *AppError) Error() string { return e.Message }

func validate() error {
    var err *AppError
    return err
}

func main() {
    fmt.Println(validate() == nil)
}
```

- Что будет выведено и почему?
- Исправьте `validate`, не меняя её return type.
- Что случится, если вызвать `validate().Error()` с текущей реализацией?

### 13. Embedding — не наследование

```go
type Logger struct{}
func (Logger) Name() string { return "logger" }

type Service struct{ Logger }
func (Service) Name() string { return "service" }
```

- Что вернут `Service{}.Name()` и `Service{}.Logger.Name()`?
- Удалите метод Service: изменится ли его method set?
- Можно ли передать `Service` туда, где требуется конкретный `Logger`, без явного обращения к полю?

### 14. Interface слишком широкий

```go
type UserRepository interface {
    Get(context.Context, int64) (User, error)
    Save(context.Context, User) error
    Delete(context.Context, int64) error
    List(context.Context) ([]User, error)
}
```

Функции нужен только `Get`.

- Перенесите минимальный интерфейс к consumer.
- Объясните, как это упростит fake в тесте.
- Измените функцию так, чтобы она принимала поведение, а не конкретную БД.

## Ошибки

### 15. Потерянная цепочка ошибок

```go
if err != nil {
    return fmt.Errorf("load user: %v", err)
}
```

- Почему `errors.Is` не найдёт исходную ошибку?
- Измените format verb.
- В каком случае намеренно не стоит раскрывать внутреннюю ошибку caller?

### 16. Ошибка Close

Функция создаёт файл, кодирует в него данные и возвращает только ошибку encoder.

- Почему этого недостаточно?
- Напишите вариант, который не теряет ошибку `Close`.
- Как аккуратно объединить ошибку записи и ошибку закрытия, не маскируя первую?

## Context и каналы отмены

### 17. Context принят, но потерян

```go
func fetch(ctx context.Context, client *http.Client, url string) error {
    req, err := http.NewRequest(http.MethodGet, url, nil)
    if err != nil {
        return err
    }
    _, err = client.Do(req)
    return err
}
```

- Почему отмена `ctx` не остановит запрос?
- Исправьте создание request.
- Добавьте корректное закрытие body и различение transport error и HTTP error.

### 18. Утечка из-за забытого cancel

```go
func call(parent context.Context) error {
    ctx, _ := context.WithTimeout(parent, time.Second)
    return dependency(ctx)
}
```

- Какой ресурс удерживается дольше необходимого?
- Исправьте функцию.
- Что произойдёт, если parent завершится раньше одной секунды?

### 19. Goroutine не умеет останавливаться

```go
func generate() <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for i := 0; ; i++ {
            out <- i
        }
    }()
    return out
}

func main() {
    for n := range generate() {
        fmt.Println(n)
        if n == 2 {
            break
        }
    }
}
```

- Какая goroutine остаётся заблокированной и где?
- Добавьте `context.Context`, чтобы caller отменял generator.
- Оберните send в `select`, иначе найдите сценарий, где отмена всё ещё не помогает.

### 20. Отправка вместо закрытия

```go
done := make(chan struct{})
for i := 0; i < 3; i++ {
    go worker(done)
}
done <- struct{}{}
```

- Сколько workers гарантированно получат сигнал?
- Замените одну операцию так, чтобы сигнал стал broadcast.
- Кто должен выполнять эту операцию и почему workers не должны конкурировать за неё?

### 21. Двойное закрытие

```go
func worker(done chan struct{}) {
    defer close(done)
    // work
}
```

Два workers получают один `done`.

- Что может произойти?
- Разделите signal отмены и signal завершения.
- Реализуйте ожидание двух workers через `WaitGroup` и закрытие results одним coordinator.

### 22. Таймаут, который не отменяет работу

```go
func resultWithTimeout() error {
    result := make(chan error)
    go func() {
        result <- slowOperation()
    }()

    select {
    case err := <-result:
        return err
    case <-time.After(100 * time.Millisecond):
        return context.DeadlineExceeded
    }
}
```

- Почему после timeout goroutine может утечь?
- Исправьте минимум только buffer-ом и объясните ограничение такого исправления.
- Затем измените `slowOperation`, чтобы она принимала context и реально прекращала работу.

### 23. Parent и child

Создайте `parent`, два дочерних context и один grandchild.

- Докажите тестом, что отмена parent отменяет всех descendants.
- Докажите, что отмена одного child не отменяет parent и sibling.
- Добавьте разные deadlines и проверьте `Err`.
- Повторите с `WithCancelCause` и сравните `Err` с `Cause`.

### 24. Кто владеет channel?

```go
func merge(a, b <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        for v := range a {
            out <- v
        }
    }()
    go func() {
        for v := range b {
            out <- v
        }
    }()
    return out
}
```

- Почему `range` по результату никогда не завершится?
- Почему ни одна из двух sender goroutines не должна самостоятельно закрывать `out`?
- Добавьте `WaitGroup` и coordinator, который закрывает `out` после обоих senders.
- Добавьте отмену, защищающую каждый send.

## Конкурентность

### 25. Race на map

```go
var counts = map[string]int{}

func inc(key string) {
    counts[key]++
}
```

- Запустите `inc` из многих goroutines и проверьте `-race`.
- Исправьте через Mutex.
- Затем переделайте на одну goroutine-владельца state и channel команд.
- Сравните сложность остановки обоих вариантов.

### 26. WaitGroup добавлен слишком поздно

```go
for i := 0; i < 10; i++ {
    go func() {
        wg.Add(1)
        defer wg.Done()
        work()
    }()
}
wg.Wait()
```

- Почему `Wait` может завершиться раньше workers?
- Перенесите одну операцию в правильное место.
- Сделайте так, чтобы panic в `work` не оставила счётчик незавершённым, и обсудите, должен ли worker вообще recover.

### 27. Busy loop в select

```go
for {
    select {
    case job := <-jobs:
        handle(job)
    default:
    }
}
```

- Почему цикл расходует CPU без работы?
- Уберите busy waiting и добавьте отмену.
- Что произойдёт после закрытия `jobs`, если получать значение без `ok`?

### 28. Закрытый channel всегда готов

```go
for {
    select {
    case v, ok := <-updates:
        if !ok {
            fmt.Println("closed")
        } else {
            fmt.Println(v)
        }
    case <-ticker.C:
        fmt.Println("tick")
    }
}
```

- Что произойдёт после закрытия `updates`?
- Сделайте case неактивным после закрытия, не завершая весь цикл.
- Объясните, почему присваивание channel переменной значения `nil` помогает.

### 29. Worker pool и ранняя ошибка

Есть пять workers. При первой ошибке нужно остановить выдачу jobs, отменить активную работу, дождаться workers и вернуть первую содержательную причину.

- Спроектируйте решение с `context.WithCancelCause`.
- Определите, кто закрывает jobs и results.
- Не допустите блокировки worker при отправке результата после ухода consumer.
- Проверьте тестом, что все goroutines завершились.

### 30. Ограничение concurrency

HTTP-handler запускает goroutine на каждый из 1000 элементов и одновременно обращается к зависимости, выдерживающей только 20 запросов.

- Ограничьте число активных вызовов channel-semaphore или worker pool.
- Добавьте отмену при закрытии client request.
- Не запускайте заранее 1000 goroutines, если обработка уже отменена.
- Верните результаты в исходном порядке, хотя выполнение происходит параллельно.

## Standard library и тесты

### 31. HTTP response body

```go
resp, err := client.Get(url)
if err != nil {
    return err
}
return json.NewDecoder(resp.Body).Decode(&dst)
```

- Какие проверки и cleanup отсутствуют?
- Обработайте status code и ограничьте размер body.
- Объясните, почему один общий `http.Client` лучше нового client на каждый запрос.

### 32. Scanner и длинная строка

Программа читает access log через `bufio.Scanner` и внезапно останавливается на очень длинной строке.

- Какую ошибку нужно проверить после цикла?
- Как увеличить допустимый token?
- Когда лучше перейти на `bufio.Reader`?

### 33. Хрупкий concurrent test

```go
go service.Refresh()
time.Sleep(10 * time.Millisecond)
if !service.Ready() {
    t.Fatal("not ready")
}
```

- Почему тест может падать только в CI?
- Добавьте явный completion signal или polling с deadline.
- Запустите с `-race` и исправьте доступ к `Ready`.

### 34. Ошибка из test goroutine

```go
go func() {
    if err := work(); err != nil {
        t.Fatal(err)
    }
}()
```

- Почему так делать нельзя?
- Передайте error основной test goroutine.
- Добавьте timeout, чтобы тест завершился при зависшем `work`.

## Итоговые проектные задачи

### 35. Graceful HTTP server

Соберите минимальный server, который:

- получает root context через `signal.NotifyContext`;
- имеет read/write/idle timeouts;
- передаёт request context в service и storage;
- при `SIGTERM` перестаёт принимать трафик и ждёт активные requests;
- имеет общий предел остановки;
- не теряет ошибку `ListenAndServe`.

Затем намеренно замените request context на `context.Background` в storage и покажите тестом или логами, что работа переживает отмену клиента.

### 36. Cancelable pipeline

Постройте pipeline: generator → validator → worker pool → collector.

Требования:

- первая validation error отменяет pipeline с причиной;
- каждый send и receive может завершиться по context;
- все output channels закрываются только их senders/coordinator;
- нет goroutine leaks при раннем выходе collector;
- тест проходит с `-race` и имеет deadline.

