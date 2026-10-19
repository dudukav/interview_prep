# Конкурентность

Собеседовательные нюансы: [состояния channels, ownership, select, leaks, Mutex, atomics и races](../interview-traps.md#5-goroutines-channels-и-shared-state).

## Concurrency и parallelism

Concurrency — организация нескольких независимо продвигающихся задач. Parallelism — их физическое выполнение одновременно. Goroutines дают concurrency; runtime планирует их на OS threads и может выполнять параллельно на нескольких CPU.

Goroutine дешевле OS thread, но не бесплатна. Она имеет stack, удерживает ссылки на данные и требует способа завершения. Для каждой запущенной goroutine нужно понимать, кто ждёт её результат и что её остановит.

## Happens-before и data race

Data race возникает, когда несколько goroutines одновременно обращаются к одной памяти, хотя бы одна запись, и нет синхронизации. Результат не просто «последний writer победил»: программа имеет некорректную конкурентную семантику.

Синхронизацию создают channel operations, mutexes, atomics и другие примитивы. `time.Sleep` и надежда на порядок scheduler синхронизацией не являются.

Запускать тесты следует с `go test -race ./...`. Race detector находит только гонки, проявившиеся в конкретном выполнении.

## Channels

Channel передаёт значения определённого типа и синхронизирует goroutines.

Небуферизованный channel завершает send, когда receiver готов принять значение. Буферизованный позволяет send завершиться, пока есть свободное место. Буфер меняет момент блокировки, но не должен использоваться для случайного скрытия deadlock.

```go
ch := make(chan int)    // unbuffered
jobs := make(chan Job, 10)
```

## Закрытие channel

`close(ch)` сообщает, что новых значений не будет. Receiver сначала получает buffered values, затем немедленно получает zero value с `ok == false`:

```go
value, ok := <-ch
```

`range ch` завершается после закрытия и опустошения. Send в closed channel вызывает panic. Повторный close вызывает panic. Получение из nil channel блокируется навсегда; closed channel и nil channel — разные состояния.

Обычно channel закрывает sender, который знает, что отправок больше не будет. Receiver не обязан закрывать channel.

## select

`select` ожидает одну из готовых channel operations. Если готовы несколько, выбирается псевдослучайная. `default` делает операцию неблокирующей, но цикл с `default` способен сжечь CPU.

```go
select {
case job := <-jobs:
    handle(job)
case <-ctx.Done():
    return ctx.Err()
}
```

Channel можно временно выключить из `select`, присвоив ему `nil`: case с nil channel никогда не готов.

## Pipeline

Pipeline состоит из stages, соединённых channels. Каждый stage должен:

- завершиться после закрытия input или отмены;
- закрыть собственный output, когда отправок больше нет;
- не блокироваться навсегда, если downstream перестал читать.

Send часто оборачивают в `select` с отменой:

```go
select {
case out <- value:
case <-ctx.Done():
    return
}
```

## Worker pool

Worker pool ограничивает concurrency. Несколько workers читают общий jobs channel, обрабатывают задания и отправляют результаты. Владелец:

1. запускает workers;
2. передаёт задания;
3. закрывает jobs;
4. ждёт workers через `WaitGroup`;
5. закрывает results после завершения всех senders.

Закрыть results раньше — panic при send. Никогда не закрыть — consumer может зависнуть на `range`.

## Mutex

`sync.Mutex` защищает инвариант shared state. Все обращения к защищаемым полям должны соблюдать один протокол lock. `defer mu.Unlock()` безопасно рядом с `Lock`, но большой critical section снижает concurrency.

`RWMutex` допускает несколько readers, но полезен только при измеренной read-heavy нагрузке и достаточно долгих операциях. Он сложнее и не автоматически быстрее обычного Mutex.

Mutex нельзя копировать после первого использования. Поэтому structures с mutex обычно передают по pointer.

## WaitGroup и Once

`WaitGroup` ждёт завершения набора goroutines. Счётчик увеличивают до запуска goroutine, иначе `Wait` может завершиться слишком рано. Каждая запущенная единица должна вызвать `Done` ровно один раз.

`sync.Once` выполняет функцию один раз. Если функция panic, Once всё равно считается использованным. Для инициализации, способной вернуть ошибку или потребовать retry, нужна отдельная модель состояния.

## Atomics

Atomic operations подходят для небольшого независимого состояния: counter или flag. Если несколько полей образуют общий инвариант, mutex обычно понятнее. Смешивать atomic и обычный доступ к одной переменной нельзя.

## Goroutine leak

Типичные причины:

- send без receiver;
- receive из channel, который никто не закроет;
- network call без deadline;
- ticker не остановлен;
- consumer прекратил чтение pipeline без отмены upstream;
- worker не слушает сигнал остановки.

Утечку ищут по растущему числу goroutines, goroutine profile и stack traces.

## Вопросы для самопроверки

1. Чем concurrency отличается от parallelism?
2. Какие события создают happens-before?
3. Чем nil channel отличается от closed channel?
4. Кто должен закрывать results в worker pool?
5. Почему buffer не является универсальным лечением deadlock?
6. Когда Mutex проще channel?
7. Почему `WaitGroup.Add` делают до `go`?
8. Как downstream сообщает pipeline, что больше не читает?

