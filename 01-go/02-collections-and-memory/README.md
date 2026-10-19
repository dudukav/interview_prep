# Коллекции, строки и память

Собеседовательные нюансы: [backing arrays, nil-значения, map, Unicode, JSON и удержание памяти](../interview-traps.md#2-slices-maps-strings-и-память).

## Array и slice

Array `[N]T` содержит ровно `N` элементов, а длина входит в тип. Присваивание array копирует все элементы.

Slice `[]T` — descriptor над backing array. Упрощённо он содержит pointer, length и capacity:

```text
slice → [ accessible elements | spare capacity ]
         <------ len ------>
         <----------- cap ------------>
```

Копирование slice копирует descriptor, а не элементы. Поэтому два slices могут видеть один backing array.

```go
a := []int{1, 2, 3}
b := a[:2]
b[0] = 9
fmt.Println(a) // [9 2 3]
```

## append

`append` возвращает новый slice descriptor. Если capacity хватает, данные добавляются в прежний array. Если нет — выделяется новый array, элементы копируются, и slices перестают разделять будущие изменения.

Результат `append` всегда нужно присваивать:

```go
s = append(s, value)
```

Не следует передавать subslice во внешний код, если он не должен изменять или удерживать большой исходный array. Для независимой копии используют `copy` или `slices.Clone`.

## nil и пустой slice

```go
var a []int       // nil, len=0
b := []int{}      // non-nil, len=0
c := make([]int, 0)
```

В большинство операций они ведут себя одинаково: можно `range`, `append`, получить `len`. Но сравнение с `nil` и некоторые форматы сериализации различаются. Slice нельзя сравнивать друг с другом, кроме сравнения с `nil`.

## Map

Map хранит пары key-value. Key должен быть comparable. Чтение отсутствующего ключа возвращает zero value, поэтому для различения используют comma-ok:

```go
value, ok := m[key]
```

Читать из nil map можно, записывать — panic. Обычная map небезопасна для конкурентной записи и для чтения одновременно с записью. Порядок обхода специально не гарантирован.

Struct, используемая как key, сравнивается по всем полям. Slice, map и function не comparable и не могут быть ключом.

## String, bytes и runes

String — неизменяемая последовательность bytes, обычно содержащая UTF-8. `len(s)` возвращает число bytes, не символов. Индексирование даёт byte. `range` декодирует UTF-8 и возвращает byte offset и rune.

```go
s := "Go界"
fmt.Println(len(s))         // 5 bytes
fmt.Println(len([]rune(s))) // 3 runes
```

Rune — alias `int32`, представляющий Unicode code point. Видимый пользователю grapheme может состоять из нескольких runes, поэтому даже `[]rune` не всегда равно числу визуальных символов.

Преобразование `string` ↔ `[]byte` обычно создаёт копию. Для построения строки используют `strings.Builder` или `bytes.Buffer`, а не многократную конкатенацию в большом цикле.

## Struct

Struct объединяет поля. Присваивание struct копирует поля. Struct comparable, только если comparable все её поля. Embedded field продвигает поля и методы, но не создаёт наследование.

Экспортируемые поля сериализуются стандартным `encoding/json`. Tags меняют имя и опции:

```go
type User struct {
    ID    int64  `json:"id"`
    Email string `json:"email"`
}
```

## JSON

`json.Marshal` кодирует значение, `json.Unmarshal` декодирует в pointer. Неизвестные поля по умолчанию игнорируются. Для строгого внешнего API можно использовать `Decoder.DisallowUnknownFields`.

`omitempty` пропускает поле при empty value, но это не всегда позволяет различить «не передано» и «передано zero value». Для PATCH-модели иногда нужны pointers или отдельные optional-типы.

Числа при декодировании в `any` по умолчанию становятся `float64`. Для точного контракта лучше декодировать в struct.

## Stack, heap и escape analysis

Compiler решает, где разместить значение. Если оно должно жить дольше stack frame или доступно из goroutine/closure, оно может escape в heap. Это решение реализации, а не часть семантики языка.

Heap очищает garbage collector. GC освобождает недостижимые объекты, но не закрывает файлы, sockets и response bodies. Такие ресурсы закрываются явно, обычно через `defer` сразу после успешного открытия.

## Частые удержания памяти

- маленький subslice удерживает большой backing array;
- map после удаления элементов может сохранить выделенную внутреннюю память;
- goroutine удерживает значения, доступные с её stack;
- global cache без лимита растёт бесконечно;
- `time.After` или timers в горячем цикле создают лишние объекты.

## Вопросы для самопроверки

1. Что копируется при присваивании slice?
2. Когда `append` меняет исходный backing array?
3. Чем nil slice отличается от nil map?
4. Почему `len(string)` не равно числу символов?
5. Какие типы могут быть map key?
6. Что означает escape в heap?
7. Почему GC не заменяет `Close`?

