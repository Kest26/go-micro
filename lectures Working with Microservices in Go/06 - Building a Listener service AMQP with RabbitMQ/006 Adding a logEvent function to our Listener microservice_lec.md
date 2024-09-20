## Добавление функции `logEvent` в наш Listener микросервис

---

### Завершение функции `logEvent` в `consumer.go`

Итак, давайте завершим реализацию функции `logEvent`, которая находится в файле `consumer.go` внутри папки `event` в нашем Listener микросервисе. Наша задача состоит в том, чтобы записывать событие, и мы уже начали этот процесс.

### Копирование логики из Broker сервиса

Перейдем в Broker сервис и найдем его обработчики в файле `handlers.go`, расположенном в папке `cmd/api`. Там мы найдем функцию `logItem`, которая отвечает за логирование событий. Мы скопируем содержимое этой функции и вставим его в нашу функцию `logEvent` в `consumer.go`, после чего внесем необходимые изменения.

#### Пример кода из Broker сервиса:

```go
func (app *Config) logItem(w http.ResponseWriter, entry LogPayload) {
    jsonData, _ := json.MarshalIndent(entry, "", "\t")

    logServiceURL := "http://logger-service/log"

    request, err := http.NewRequest("POST", logServiceURL, bytes.NewBuffer(jsonData))
    if err != nil {
        app.errorJSON(w, err)
        return
    }

    request.Header.Set("Content-Type", "application/json")

    client := &http.Client{}
    response, err := client.Do(request)
    if err != nil {
        app.errorJSON(w, err)
        return
    }
    defer response.Body.Close()

    if response.StatusCode != http.StatusAccepted {
        app.errorJSON(w, err)
        return
    }

    var payload jsonResponse
    payload.Error = false
    payload.Message = "logged"

    app.writeJSON(w, http.StatusAccepted, payload)
}
```

### Вставка и модификация кода в `consumer.go`

Теперь вернемся в `consumer.go` и вставим скопированный код в функцию `logEvent`. После вставки необходимо внести следующие изменения:

1. **Импорт необходимых пакетов**:
    - Добавим импорт пакета `http`, чтобы использовать функции для HTTP-запросов.
    - Добавим импорт пакета `bytes` вручную в секцию импорта.

    ```go
    import (
        "bytes"
        "encoding/json"
        "fmt"
        "log"
        "net/http"

        amqp "github.com/rabbitmq/amqp091-go"
    )
    ```

2. **Удаление ненужных частей кода**:
    - Удалим части, связанные с обработкой JSON ошибок, так как в нашем случае это не требуется.
    - Удалим упоминания о несуществующем `app.errorJSON`.

3. **Возврат ошибок**:
    - Вместо `app.errorJSON` будем просто возвращать ошибки.

#### Обновленная функция `logEvent`:

```go
func logEvent(entry Payload) error {
    jsonData, _ := json.MarshalIndent(entry, "", "\t")

    logServiceURL := "http://logger-service/log"

    request, err := http.NewRequest("POST", logServiceURL, bytes.NewBuffer(jsonData))
    if err != nil {
        return err
    }

    request.Header.Set("Content-Type", "application/json")

    client := &http.Client{}
    response, err := client.Do(request)
    if err != nil {
        return err
    }
    defer response.Body.Close()

    if response.StatusCode != http.StatusAccepted {
        return fmt.Errorf("received non-accepted status code: %d", response.StatusCode)
    }

    return nil
}
```

### Объяснение изменений

- **Импорт пакетов**:
    - `net/http`: Необходим для создания и отправки HTTP-запросов.
    - `bytes`: Используется для создания буфера из JSON-данных.

- **Создание HTTP-запроса**:
    ```go
    request, err := http.NewRequest("POST", logServiceURL, bytes.NewBuffer(jsonData))
    if err != nil {
        return err
    }
    ```
    Создаем новый HTTP POST запрос к сервису логирования с JSON-данными в теле запроса.

- **Установка заголовков**:
    ```go
    request.Header.Set("Content-Type", "application/json")
    ```
    Устанавливаем заголовок `Content-Type` как `application/json` для корректной передачи данных.

- **Отправка запроса**:
    ```go
    client := &http.Client{}
    response, err := client.Do(request)
    if err != nil {
        return err
    }
    defer response.Body.Close()
    ```
    Создаем HTTP клиент и отправляем запрос. После получения ответа закрываем тело ответа с помощью `defer`.

- **Проверка статуса ответа**:
    ```go
    if response.StatusCode != http.StatusAccepted {
        return fmt.Errorf("received non-accepted status code: %d", response.StatusCode)
    }
    ```
    Проверяем, что статус ответа равен `202 Accepted`. В противном случае возвращаем ошибку.

- **Возврат ошибки или успешное завершение**:
    ```go
    return nil
    ```
    Если все прошло успешно, возвращаем `nil`, указывая на отсутствие ошибок.

### Обработка дублирования кода

Вы, вероятно, заметили, что мы дублируем код из Broker сервиса в Listener микросервисе. Это действительно выглядит как дублирование, но помните, что каждый микросервис должен быть автономным и самодостаточным. В дальнейшем, мы будем улучшать это, используя RPC вместо прямых HTTP-запросов, что позволит избежать дублирования и повысить эффективность взаимодействия между сервисами.

### Завершение изменений в `consumer.go`

Теперь, если все изменения внесены правильно, наша функция `logEvent` готова к использованию. Однако, чтобы убедиться, что всё работает корректно, нам необходимо внести ещё несколько изменений:

1. **Модификация Broker сервиса**:
    - Нам нужно изменить Broker сервис, чтобы он отправлял сообщения в очередь, которую слушает Listener сервис.
  
2. **Обновление `main.go` в Listener сервисе**:
    - В файле `main.go` Listener сервиса необходимо реализовать логику, обозначенную в комментариях, чтобы настроить и запустить потребление сообщений из очереди.

Эти изменения будут рассмотрены в следующей лекции.

### Итог

На этом этапе мы завершили добавление функции `logEvent` в наш Listener микросервис. Мы скопировали и адаптировали код из Broker сервиса, чтобы обеспечить логирование событий, поступающих в очередь. В следующей лекции мы продолжим настройку взаимодействия между сервисами и оптимизируем процесс логирования, перейдя от прямых HTTP-запросов к использованию RPC.

---

## Полный код файлов после изменений

### `listener-service/event/consumer.go`

```go
package event

import (
    "bytes"
    "encoding/json"
    "fmt"
    "log"
    "net/http"

    amqp "github.com/rabbitmq/amqp091-go"
)

type Consumer struct {
    conn      *amqp.Connection
    queueName string
}

func NewConsumer(conn *amqp.Connection) (Consumer, error) {
    consumer := Consumer{
        conn: conn,
    }

    err := consumer.setup()
    if err != nil {
        return Consumer{}, err
    }

    return consumer, nil
}

func (consumer *Consumer) setup() error {
    channel, err := consumer.conn.Channel()
    if err != nil {
        return err
    }

    return declareExchange(channel)
}

type Payload struct {
    Name string `json:"name"`
    Data string `json:"data"`
}

func (consumer *Consumer) Listen(topics []string) error {
    ch, err := consumer.conn.Channel()
    if err != nil {
        return err
    }
    defer ch.Close()

    q, err := declareRandomQueue(ch)
    if err != nil {
        return err
    }

    for _, s := range topics {
        err := ch.QueueBind(
            q.Name,        // имя очереди
