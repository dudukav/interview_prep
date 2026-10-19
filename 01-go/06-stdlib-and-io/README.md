# Стандартная библиотека и I/O

Собеседовательные нюансы: [`io.Reader`, HTTP, time, JSON, environment и packages](../interview-traps.md#6-стандартная-библиотека-и-io).

## Интерфейсы io.Reader и io.Writer

`io.Reader` заполняет переданный buffer и возвращает число bytes и error:

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}
```

Reader может вернуть одновременно `n > 0` и `err == io.EOF`; сначала нужно обработать bytes, затем error. Один `Read` не обязан заполнить buffer. Для чтения точного числа bytes существует `io.ReadFull`, для копирования потока — `io.Copy`.

`io.Writer` обязан сообщить число принятых bytes. `io.ReadAll` удобен только для заведомо ограниченного входа: иначе приложение может исчерпать память. Размер внешнего body ограничивают, например, через `io.LimitReader` или HTTP-механизмы.

## Закрытие ресурсов

`Close` освобождает descriptor или возвращает ресурс в pool. После успешного открытия close обычно планируют сразу:

```go
f, err := os.Open(name)
if err != nil {
    return err
}
defer f.Close()
```

В большом цикле `defer` откладывается до выхода из всей функции, поэтому одну итерацию лучше вынести в отдельную функцию. Для записи важно обработать ошибку `Close`: buffered данные могут окончательно записываться именно там.

## bufio

Buffer уменьшает число системных вызовов. `bufio.Reader` и `Writer` оборачивают поток. `Writer.Flush` обязателен. `Scanner` удобен для строк и tokens, но имеет ограничение размера token; его можно изменить через `Scanner.Buffer` или использовать Reader.

## encoding/json

`Encoder` и `Decoder` работают с потоками, `Marshal`/`Unmarshal` — с готовыми byte slices. Для HTTP полезно отделять transport DTO от domain model.

Важные моменты:

- сериализуются экспортируемые поля;
- неизвестные поля по умолчанию игнорируются;
- числа в `any` становятся `float64` без `UseNumber`;
- `omitempty` зависит от empty value, а не бизнес-смысла;
- decoder способен прочитать несколько JSON values, поэтому для API нужно контролировать лишний ввод.

## time

Duration — число наносекунд с удобными constants. Timer срабатывает один раз, Ticker периодически. Ticker нужно останавливать, когда он больше не нужен.

Для дедлайнов используют `time.NewTimer` или context, а не `time.Sleep` в отдельной goroutine. Монотонная часть `time.Time` помогает измерять длительность внутри процесса; сериализация сохраняет календарное время, но не монотонную часть.

В БД и API следует явно договориться о timezone, обычно хранить timestamp в UTC и преобразовывать на границе отображения.

## net/http server

`http.Handler` имеет один метод:

```go
ServeHTTP(http.ResponseWriter, *http.Request)
```

Handler должен уважать `r.Context()`. Middleware оборачивает следующий Handler и подходит для logging, authentication, recovery и метрик.

Для production server задают timeouts и выполняют graceful shutdown. Нельзя без ограничений читать body или доверять входным headers. После записи headers status code изменить нельзя; если вызвать `Write` первым, будет отправлен `200`.

## net/http client

Следует переиспользовать `http.Client`: он содержит Transport с pool соединений. `http.DefaultClient` не имеет общего timeout и не всегда подходит.

```go
req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
resp, err := client.Do(req)
```

Успешный `Do` не означает HTTP success: `404` и `500` приходят без transport error. `resp.Body` нужно закрыть; для переиспользования connection обычно также дочитать до конца в разумных пределах.

## fmt и логирование

`fmt.Errorf` с `%w` оборачивает error. Для структурированных production logs лучше logger с полями. Нельзя логировать секреты и один error на каждом слое без добавления нового смысла.

## os и сигналы

Конфигурацию часто получают из environment. Отсутствующие или неверные обязательные значения проверяют при старте.

`signal.NotifyContext` удобно создаёт context, отменяемый при `SIGINT`/`SIGTERM`. Этот context передают server, workers и consumers, чтобы координировать остановку.

## Пакеты и internal

Package должен иметь узкую ответственность и короткое имя. Import cycles запрещены и часто указывают на неправильное направление зависимостей.

Каталог `internal` ограничивает импорт кодом из родительского дерева. Package `main` связывает dependencies; domain logic не должна импортировать transport только ради запуска приложения.

## Reflection и unsafe

Reflection позволяет исследовать dynamic types и values и используется в serializers, validators и frameworks. Цена — сложность, runtime-проверки и менее ясный код. Для обычной бизнес-логики interfaces и generics предпочтительнее.

`unsafe` обходит гарантии type system и зависит от представления памяти. Для junior backend это тема узнавания, а не повседневный инструмент.

## Вопросы для самопроверки

1. Может ли Reader вернуть данные вместе с `io.EOF`?
2. Почему `io.ReadAll` опасен для внешнего body?
3. Когда ошибка может проявиться в `Close` или `Flush`?
4. Почему следует переиспользовать `http.Client`?
5. Чем transport error отличается от HTTP `500`?
6. Зачем Handler использует `r.Context()`?
7. Что ограничивает каталог `internal`?

