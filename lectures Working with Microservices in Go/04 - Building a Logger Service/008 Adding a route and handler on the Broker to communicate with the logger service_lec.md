## Добавление маршрута и обработчика в `Broker-service` для связи с сервисом логирования `logger-service`

В этой лекции мы рассмотрим, как добавить маршрут и обработчик в брокерный микросервис для взаимодействия с сервисом логирования. Изначально каждый микросервис будет напрямую взаимодействовать с сервисом логирования, но для целей тестирования мы настроим связь через брокер. Рассмотрим поэтапно, какие изменения необходимо внести в код.

### 1. Добавление новой структуры данных

Для начала откроем файл `handlers.go` в `broker-service` и добавим новую структуру данных `LogPayload`, которая будет использоваться для передачи логов.

```go
type LogPayload struct {
    Name string `json:"name"`
    Data string `json:"data"`
}
```

Эта структура содержит два поля: `Name` и `Data`, оба типа `string`. Мы добавляем её в структуру `RequestPayload`, чтобы она могла обрабатываться как часть общего запроса.

```go
type RequestPayload struct {
    Action string      `json:"action"`
    Auth   AuthPayload `json:"auth,omitempty"`
    Log    LogPayload  `json:"log,omitempty"`
}
```

### 2. Обновление метода `HandleSubmission`

Метод `HandleSubmission` уже обрабатывает запросы, в которых действие (`Action`) равно `"auth"`. Теперь добавим поддержку нового действия `"log"`.

```go
func (app *Config) HandleSubmission(w http.ResponseWriter, r *http.Request) {
    var requestPayload RequestPayload

    err := app.readJSON(w, r, &requestPayload)
    if err != nil {
        app.errorJSON(w, err)
        return
    }

    switch requestPayload.Action {
    case "auth":
        app.authenticate(w, requestPayload.Auth)
    case "log":
        app.logItem(w, requestPayload.Log)
    default:
        app.errorJSON(w, errors.New("unknown action"))
    }
}
```

Теперь при передаче действия `"log"` запрос будет направлен в новый метод `logItem`, который мы определим далее.

### 3. Реализация метода `logItem`

Метод `logItem` будет отправлять данные логирования в наш сервис логирования. Сначала он сериализует данные логов в JSON, затем формирует HTTP-запрос и отправляет его на соответствующий эндпоинт сервиса логирования.

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
        app.errorJSON(w, errors.New("error logging"))
        return
    }

    var payload jsonResponse
    payload.Error = false
    payload.Message = "logged"

    app.writeJSON(w, http.StatusAccepted, payload)
}
```

Этот метод формирует HTTP-запрос с типом POST и отправляет его на URL сервиса логирования (`logger-service/log`). Если запрос выполняется успешно и возвращает статус `HTTP.StatusAccepted`, сервис считает, что лог был успешно обработан.

### 4. Добавление нового маршрута в `routes.go`

Теперь необходимо убедиться, что новый маршрут обработчика добавлен в файл маршрутов `routes.go`.

```go
mux.Post("/handle", app.HandleSubmission)
```

Маршрут остаётся прежним, поскольку мы расширили существующий метод `HandleSubmission`, добавив возможность обработки действий `"log"`.

### 5. Проверка работы

Теперь, когда изменения внесены, можно пересобрать сервисы и проверить их работу, выполнив команду:

```bash
make up_build
```

Это обновит и перезапустит все контейнеры с актуальным кодом. После этого можно протестировать отправку логов с фронтенда через брокерный сервис.

## Заключение

Мы добавили поддержку нового действия `"log"` в брокерный микросервис, что позволяет передавать данные логов в сервис логирования. В следующей лекции мы настроим фронтенд для взаимодействия с этим функционалом.

---

Теперь вы готовы к следующему этапу настройки взаимодействия между сервисами!
