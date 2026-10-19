# Методы, интерфейсы и generics

Собеседовательные нюансы: [method sets, typed nil, сравнение interfaces, embedding и generics](../interview-traps.md#3-методы-интерфейсы-и-generics).

## Методы

Метод — функция с receiver:

```go
type Counter int

func (c Counter) Value() int { return int(c) }
func (c *Counter) Inc()      { *c++ }
```

Value receiver получает копию. Pointer receiver может менять исходное значение и избегает копирования большой структуры. Для одного типа обычно выбирают согласованный стиль, особенно если хотя бы один метод требует pointer.

Compiler иногда автоматически вставляет `&` или `*` при обычном вызове метода у addressable value. Это удобство вызова не меняет method set типа.

## Method set

Упрощённо:

- method set `T` содержит методы с receiver `T`;
- method set `*T` содержит методы с receiver `T` и `*T`.

Поэтому если интерфейс требует метод, объявленный только на `*T`, значение `T` интерфейс не реализует, а `*T` реализует.

## Интерфейс

Интерфейс задаёт набор методов. Тип реализует его неявно — отдельного `implements` нет. Это уменьшает связанность: потребитель может описать минимальное необходимое поведение рядом с местом использования.

```go
type Store interface {
    Get(ctx context.Context, id int64) (User, error)
}
```

Хороший интерфейс обычно маленький и появляется из потребности consumer, а не создаётся «на всякий случай» для каждой структуры.

## Значение интерфейса и typed nil

Интерфейс conceptually содержит dynamic type и dynamic value. Он равен `nil`, только если отсутствуют оба.

```go
var p *MyError = nil
var err error = p
fmt.Println(err == nil) // false
```

В `err` есть dynamic type `*MyError`, поэтому интерфейс не nil. Метод `Error` может даже panic, если разыменует nil receiver.

## Type assertion и type switch

```go
value, ok := x.(Concrete)
```

Assertion извлекает значение конкретного типа или проверяет дополнительный интерфейс. Форма без `ok` вызывает panic при несовпадении.

Type switch обрабатывает несколько dynamic types. Его стоит применять, когда поведение действительно зависит от закрытого набора типов; часто обычный полиморфный метод проще.

## Embedding

Embedded field продвигает методы и поля:

```go
type Service struct {
    Logger
}
```

Это композиция. `Service` не является подтипом `Logger`, а конфликтующие promoted methods нужно разрешать явно. Публичное embedding также делает методы embedded type частью API внешнего типа.

## Интерфейс error

`error` — интерфейс с методом `Error() string`. Конкретная ошибка может нести структурированные данные и участвовать в chain через `Unwrap`.

Потребитель обычно проверяет семантику через `errors.Is` или тип через `errors.As`, а не сравнивает текст.

## Generics

Type parameter ограничивается constraint:

```go
func Contains[T comparable](items []T, target T) bool {
    for _, item := range items {
        if item == target {
            return true
        }
    }
    return false
}
```

Generics полезны, когда один алгоритм действительно одинаков для нескольких типов. Они не заменяют interfaces поведения: interface отвечает на вопрос «что значение умеет», type parameter — «для семейства каких типов компилируется функция».

Слишком абстрактный generic-код сложнее читать. Для одного-двух простых применений обычная функция может быть лучше.

## Вопросы для самопроверки

1. Чем value receiver отличается от pointer receiver?
2. Почему `T` и `*T` могут по-разному реализовывать интерфейс?
3. Почему interface с nil pointer внутри не равен `nil`?
4. Где лучше объявлять интерфейс — у producer или consumer?
5. Чем embedding отличается от наследования?
6. Когда нужен type assertion?
7. Чем generic constraint отличается от обычного interface-поля?

