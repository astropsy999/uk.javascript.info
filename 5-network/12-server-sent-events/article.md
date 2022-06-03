# Server Sent Events

Специфікація [Server-Sent Events](https://html.spec.whatwg.org/multipage/comms.html#the-eventsource-interface) визначає вбудований клас `EventSource`, який підтримує з’єднання з сервером і дозволяє отримувати від нього події.

Подібно до `WebSocket`, з’єднання є постійним.

Але є кілька важливих відмінностей:

| `WebSocket` | `EventSource` |
|-------------|---------------|
| Двонаправленність: клієнт і сервер можуть обмінюватися повідомленнями | Односпрямованність: дані надсилає лише сервер |
| Двійкові та текстові дані | Тільки текст |
| WebSocket протокол | Звичайний HTTP |

`EventSource` - це менш потужний спосіб зв’язку з сервером, ніж `WebSocket`.

Чому іноді потрібно його використовувати?

Основна причина: він простіший. Для багатьох випадків потужність `WebSocket` є дещо занадто великою.

Нам потрібно отримати потік даних із сервера: можливо, повідомлення в чаті чи ринкові ціни, чи що завгодно. `EventSource` добре це може. Також він підтримує автоматичне перепідключення, що у випадку `WebSocket` потрібно реалізовувати вручну. До того ж, це звичайний старий, а не новий  HTTP протокол.

## Отримання повідомлень

Щоб почати отримувати повідомлення, необхідно просто створити `new EventSource(url)`.

Браузер підключиться до `url` і залишить з’єднання відкритим, чекаючи подій.

Сервер повинен відповісти статусом 200 і заголовком `Content-Type: text/event-stream`, а потім зберегти з’єднання та записувати в нього повідомлення в спеціальному форматі, наприклад:

```
data: Повідомлення 1

data: Повідомлення 2

data: Повідомлення 3
data: з двох рядків
```

- Текст повідомлення йде після `data:`, пробіл після двокрапки необов’язковий.
- Повідомлення розділені подвійними розривами рядків `\n\n`.
- Щоб надіслати розрив рядка `\n`, ми можемо негайно надіслати ще одне `data:` (третє повідомлення вище).

На практиці складні повідомлення зазвичай надсилаються в кодуванні JSON. Розриви рядків кодуються як `\n` всередині них, тому багаторядкові повідомлення `data:` не потрібні.

Наприклад:

```js
data: {"user":"Тарас","message":"Перший рядок*!*\n*/!* Другий рядок"}
```

...Отже, ми можемо припустити, що одне `data:` містить рівно одне повідомлення.

Для кожного такого повідомлення генерується подія `message`:

```js
let eventSource = new EventSource("/events/subscribe");

eventSource.onmessage = function(event) {
  console.log("Нове повідомлення", event.data);
  // реєструватиме 3 рази для потоку даних, наведеного вище
};

// чи eventSource.addEventListener('message', ...)
```

### Запити з перехресних джерел

`EventSource` підтримує запити між джерелами, як-от `fetch` та будь-які інші мережеві методи. Ми можемо використовувати будь-яку URL-адресу:

```js
let source = new EventSource("https://another-site.com/events");
```

Віддалений сервер отримає заголовок `Origin` і повинен відповісти `Access-Control-Allow-Origin` щоб продовжити.

Щоб передати облікові дані, ми повинні встановити додатковий параметр `withCredentials`, наприклад:

```js
let source = new EventSource("https://another-site.com/events", {
  withCredentials: true
});
```

Будь ласка, перегляньте розділ <info:fetch-crossorigin>, щоб дізнатися більше про заголовки з перехресними джерелами.


## Повторне підключення

Після створення `new EventSource` підключається до сервера, і якщо з’єднання розривається - автоматично підключається знову.

Це дуже зручно, оскільки нам не потрібно про це дбати.

Між повторними підключеннями є невелика затримка, типово кілька секунд.

Сервер може встановити рекомендовану затримку, за допомогою `retry:` у відповіді (в мілісекундах):

```js
retry: 15000
data: Привіт, я встановив затримку для перепідключення 15 секунд
```

`retry:` може надсилатись як разом з деякими даними, так і окремим повідомленням

Браузер повинен зачекати стільки мілісекунд перед повторним підключенням. Або довше, напр. якщо браузер знає (з ОС), що на даний момент немає підключення до мережі, він може зачекати, доки з’єднання з’явиться, а потім повторити спробу.

- Якщо сервер хоче, щоб браузер припинив повторне підключення, він повинен відповісти HTTP статусом 204.
- Якщо браузер хоче закрити з’єднання, він повинен викликати `eventSource.close()`:

```js
let eventSource = new EventSource(...);

eventSource.close();
```

Крім того, не буде повторного підключення, якщо відповідь містить неправильний `Content-Type` або його статус HTTP відрізняється від 301, 307, 200 і 204. У таких випадках буде створено подію `"error"`, і браузер не підключатиметься знову.

```smart
Коли з’єднання остаточно закрите, його неможливо «відкрити». Якщо ми хочемо знову під’єднатися, доведеться створити новий `EventSource`.
```

## Message id

When a connection breaks due to network problems, either side can't be sure which messages were received, and which weren't.

To correctly resume the connection, each message should have an `id` field, like this:

```
data: Message 1
id: 1

data: Message 2
id: 2

data: Message 3
data: of two lines
id: 3
```

When a message with `id:` is received, the browser:

- Sets the property `eventSource.lastEventId` to its value.
- Upon reconnection sends the header `Last-Event-ID` with that `id`, so that the server may re-send following messages.

```smart header="Put `id:` after `data:`"
Please note: the `id` is appended below message `data` by the server, to ensure that `lastEventId` is updated after the message is received.
```

## Connection status: readyState

The `EventSource` object has `readyState` property, that has one of three values:

```js no-beautify
EventSource.CONNECTING = 0; // connecting or reconnecting
EventSource.OPEN = 1;       // connected
EventSource.CLOSED = 2;     // connection closed
```

When an object is created, or the connection is down, it's always `EventSource.CONNECTING` (equals `0`).

We can query this property to know the state of `EventSource`.

## Event types

By default `EventSource` object generates three events:

- `message` -- a message received, available as `event.data`.
- `open` -- the connection is open.
- `error` -- the connection could not be established, e.g. the server returned HTTP 500 status.

The server may specify another type of event with `event: ...` at the event start.

For example:

```
event: join
data: Bob

data: Hello

event: leave
data: Bob
```

To handle custom events, we must use `addEventListener`, not `onmessage`:

```js
eventSource.addEventListener('join', event => {
  alert(`Joined ${event.data}`);
});

eventSource.addEventListener('message', event => {
  alert(`Said: ${event.data}`);
});

eventSource.addEventListener('leave', event => {
  alert(`Left ${event.data}`);
});
```

## Full example

Here's the server that sends messages with `1`, `2`, `3`, then `bye` and breaks the connection.

Then the browser automatically reconnects.

[codetabs src="eventsource"]

## Summary

`EventSource` object automatically establishes a persistent connection and allows the server to send messages over it.

It offers:
- Automatic reconnect, with tunable `retry` timeout.
- Message ids to resume events, the last received identifier is sent in `Last-Event-ID` header upon reconnection.
- The current state is in the `readyState` property.

That makes `EventSource` a viable alternative to `WebSocket`, as the latter is more low-level and lacks such built-in features (though they can be implemented).

In many real-life applications, the power of `EventSource` is just enough.

Supported in all modern browsers (not IE).

The syntax is:

```js
let source = new EventSource(url, [credentials]);
```

The second argument has only one possible option: `{ withCredentials: true }`, it allows sending cross-origin credentials.

Overall cross-origin security is same as for `fetch` and other network methods.

### Properties of an `EventSource` object

`readyState`
: The current connection state: either `EventSource.CONNECTING (=0)`, `EventSource.OPEN (=1)` or `EventSource.CLOSED (=2)`.

`lastEventId`
: The last received `id`. Upon reconnection the browser sends it in the header `Last-Event-ID`.

### Methods

`close()`
: Closes the connection.

### Events

`message`
: Message received, the data is in `event.data`.

`open`
: The connection is established.

`error`
: In case of an error, including both lost connection (will auto-reconnect) and fatal errors. We can check `readyState` to see if the reconnection is being attempted.

The server may set a custom event name in `event:`. Such events should be handled using `addEventListener`, not `on<event>`.

### Server response format

The server sends messages, delimited by `\n\n`.

A message may have following fields:

- `data:` -- message body, a sequence of multiple `data` is interpreted as a single message, with `\n` between the parts.
- `id:` -- renews `lastEventId`, sent in `Last-Event-ID` on reconnect.
- `retry:` -- recommends a retry delay for reconnections in ms. There's no way to set it from JavaScript.
- `event:` -- event name, must precede `data:`.

A message may include one or more fields in any order, but `id:` usually goes the last.
