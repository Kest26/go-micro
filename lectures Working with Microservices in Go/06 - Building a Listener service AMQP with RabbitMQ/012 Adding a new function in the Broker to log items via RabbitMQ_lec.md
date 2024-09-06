## Добавление новой функции для логирования элементов через RabbitMQ в сервисе брокера

#### Введение

Сегодня мы продолжаем работу с сервисом брокера, сосредоточив внимание на улучшении логирования. В предыдущих лекциях логирование осуществлялось с помощью HTTP-запросов, но теперь мы хотим изменить механизм и отправлять логи через RabbitMQ. В этой лекции мы создадим новую функцию для логирования и упростим отправку сообщений в очередь RabbitMQ с помощью вспомогательной утилиты. 

### Шаг 1: Определение проблемы и подготовка изменений

Сейчас, когда в полученном запросе тип действия (`action`) равен `"log"`, система вызывает функцию `logItem`, которая отправляет лог-сообщение через POST-запрос. Наша задача — создать новую функцию для логирования через RabbitMQ и заменить вызов `logItem` на новую функцию. 

В текущем коде `handlers.go` функция `logItem` выглядит так:

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
        app.errorJSON(w, errors.New("error calling log service"))
        return
    }

    var payload jsonResponse
    payload.Error = false
    payload.Message = "logged"
    app.writeJSON(w, http.StatusAccepted, payload)
}
```

Эта функция делает HTTP-запрос к сервису логирования, но теперь наша цель — отправлять логи через RabbitMQ. Для этого создадим две новые функции.

### Шаг 2: Создание функции для логирования через RabbitMQ

Мы создадим функцию `logEventViaRabbit`, которая будет отправлять лог-сообщение в RabbitMQ. Эта функция принимает два параметра: объект `http.ResponseWriter` для отправки ответа клиенту и объект `LogPayload`, содержащий данные для логирования.

Пример кода для новой функции:

```go
func (app *Config) logEventViaRabbit(w http.ResponseWriter, l LogPayload) {
    err := app.pushToQueue(l.Name, l.Data)
    if err != nil {
        app.errorJSON(w, err)
        return
    }

    var payload jsonResponse
    payload.Error = false
    payload.Message = "logged via RabbitMQ"

    app.writeJSON(w, http.StatusAccepted, payload)
}
```

Эта функция вызывает вспомогательную функцию `pushToQueue`, которую мы создадим далее.

### Шаг 3: Создание вспомогательной функции для отправки сообщений в RabbitMQ

Теперь мы создадим утилитарную функцию `pushToQueue`, которая будет отвечать за отправку сообщений в очередь RabbitMQ. Она принимает два параметра: `name` (название события) и `msg` (сообщение, которое нужно отправить). Функция возвращает ошибку, если что-то пошло не так.

Пример кода для функции `pushToQueue`:

```go
func (app *Config) pushToQueue(name, msg string) error {
    emitter, err := event.NewEventEmitter(app.Rabbit)
    if err != nil {
        return err
    }

    payload := LogPayload{
        Name: name,
        Data: msg,
    }

    j, _ := json.MarshalIndent(&payload, "", "\t")
    err = emitter.Push(string(j), "log.INFO")
    if err != nil {
        return err
    }
    return nil
}
```

Разбор функции:
1. Мы создаём объект `emitter` с помощью функции `event.NewEventEmitter`, которая принимает подключение к RabbitMQ.
2. Далее создаём объект `payload` типа `LogPayload` и заполняем его полями `Name` и `Data`.
3. Сериализуем объект в формат JSON и отправляем его в очередь с помощью метода `Push` объекта `emitter`.

#### Шаг 4: Обновление логики обработки запросов

Теперь, когда функция для логирования готова, необходимо обновить логику обработки запросов. В функции `HandleSubmission`, где происходит выбор действия в зависимости от значения поля `action`, заменим вызов `logItem` на новую функцию `logEventViaRabbit`.

Пример кода обновлённого `switch`-блока:

```go
switch requestPayload.Action {
case "auth":
    app.authenticate(w, requestPayload.Auth)
case "log":
    app.logEventViaRabbit(w, requestPayload.Log)
case "mail":
    app.sendMail(w, requestPayload.Mail)
default:
    app.errorJSON(w, errors.New("unknown action"))
}
```

Теперь, когда в запросе приходит действие `"log"`, система будет использовать новую функцию для отправки логов через RabbitMQ.

### Шаг 5: Тестирование и завершение

Теперь, когда все необходимые изменения внесены, можно приступить к тестированию. Мы не вносили изменений в структуру отправляемых данных, поэтому фронтенд не требует правок. После сборки проекта и запуска всех сервисов через Docker Compose, можно протестировать функциональность логирования, отправив запрос. При успешной отправке сообщения в очередь, система вернёт ответ с сообщением `"logged via RabbitMQ"`.

---

Таким образом, мы добавили новую функцию для логирования через RabbitMQ, создали вспомогательную утилиту для отправки сообщений в очередь и обновили существующую логику сервиса брокера. Теперь система логирует события через RabbitMQ, что обеспечивает более гибкую и масштабируемую архитектуру.
