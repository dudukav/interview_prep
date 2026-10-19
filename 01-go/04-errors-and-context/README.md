# Ошибки, context и отмена

Собеседовательные нюансы: [error chains, Close, propagation context, Cause и каналы отмены](../interview-traps.md#4-errors-и-context).

## Ошибки как значения

Ожидаемая ошибка возвращается обычным значением:

```go
user, err := store.Get(ctx, id)
if err != nil {
    return User{}, fmt.Errorf("get user %d: %w", id, err)
}
```

Ошибка должна добавлять контекст, но сохранять исходную причину через `%w`, если caller должен её распознать. Текст ошибки обычно начинается со строчной буквы и не содержит лишнюю пунктуацию: несколько уровней wrapping сформируют одно сообщение.

## errors.Is и errors.As

Sentinel error обозначает известную категорию:

```go
var ErrNotFound = errors.New("not found")
```

`errors.Is(err, ErrNotFound)` проходит по error chain. `errors.As` находит ошибку определённого типа и извлекает её данные. Сравнивать wrapped error через `==` или анализировать текст ненадёжно.

Тип ошибки оправдан, если caller нужны поля:

```go
type ValidationError struct {
    Field string
    Err   error
}
```

Нельзя без необходимости экспортировать ошибки внутренней зависимости: это связывает внешний API с реализацией storage или transport.

## Panic

Panic подходит для нарушенного внутреннего инварианта или ошибки инициализации, после которой процесс не может продолжить работу. Неверные пользовательские данные, отсутствие строки и timeout — обычные errors.

HTTP middleware может recover, записать stack и вернуть `500`, сохранив работу процесса. После recover нельзя считать частично изменённое состояние автоматически корректным.

## Зачем нужен context.Context

Context переносит через границы API:

- сигнал отмены;
- deadline;
- request-scoped metadata.

Обычная сигнатура:

```go
func (s *Service) Get(ctx context.Context, id int64) (User, error)
```

`ctx` передают первым параметром, не хранят в struct как общее состояние и не заменяют им обязательные аргументы функции. Incoming request создаёт context, а все исходящие операции должны получить его или производный context.

## Дерево контекстов

`context.Background()` — корневой context приложения. `context.TODO()` временно обозначает место, где решение ещё не принято.

```go
ctx, cancel := context.WithTimeout(parent, 2*time.Second)
defer cancel()
```

Derived context отменяется, когда:

- вызван его `cancel`;
- отменён parent;
- истёк его deadline.

Отмена распространяется вниз, но не вверх. `cancel` нужно вызвать даже при наличии timeout: он освобождает timer и связи раньше.

## Done, Err и Cause

`ctx.Done()` возвращает channel, который закрывается при отмене. Закрытие удобно как broadcast: все receivers разблокируются.

```go
select {
case result := <-results:
    return result, nil
case <-ctx.Done():
    return Result{}, ctx.Err()
}
```

`ctx.Err()` возвращает `context.Canceled` или `context.DeadlineExceeded`. `WithCancelCause` и `context.Cause` позволяют отдельно сохранить смысловую причину отмены.

## Отмена должна быть кооперативной

Context сам не останавливает goroutine. Код должен ожидать `Done` в местах блокировки или периодически проверять `Err` в вычислительном цикле. Зависимость также должна поддерживать context: переданный, но игнорируемый `ctx` ничего не отменяет.

Неправильно:

```go
go slowWork() // не получает ctx и может утечь
```

Правильно:

```go
go func() {
    if err := slowWork(ctx); err != nil {
        // передать результат владельцу goroutine
    }
}()
```

## Context values

`WithValue` применяют для request-scoped метаданных, проходящих через API: trace ID или аутентифицированный principal. Его не используют для конфигурации, optional-параметров и зависимостей.

Key делают отдельного неэкспортируемого типа, чтобы packages не сталкивались именами. Доступ удобно закрыть type-safe функциями.

## Канал отмены без context

Внутри локального конкурентного алгоритма иногда достаточно `done <-chan struct{}`. Владелец создаёт channel и закрывает его ровно один раз:

```go
done := make(chan struct{})
defer close(done)
```

Workers слушают `<-done`. Закрытие — broadcast, отправка одного значения разбудит только одного receiver. Закрывать channel должен владелец сигнала, не каждый worker.

Для публичного или request-scoped API предпочтителен `context.Context`: он стандартно объединяет отмену, deadline и propagation.

## Частые ошибки

- создаётся `WithTimeout`, но `cancel` не вызывается;
- функция принимает `ctx`, но вызывает dependency с `context.Background()`;
- результат отправляется в channel после ухода consumer, и goroutine навсегда блокируется;
- context хранится в долгоживущем struct;
- `WithValue` используется как мешок параметров;
- отмена считается ошибкой уровня `ERROR`, хотя клиент штатно закрыл запрос;
- timeout каждого downstream-вызова длиннее оставшегося deadline родителя.

## Вопросы для самопроверки

1. Когда использовать `%w`, `errors.Is` и `errors.As`?
2. Чем panic отличается от error?
3. Почему `defer cancel()` нужен рядом с `WithTimeout`?
4. Отменяет ли context goroutine принудительно?
5. Почему `Done` закрывается, а не получает одно значение?
6. Для каких данных допустим `WithValue`?
7. Кто должен закрывать канал отмены?

