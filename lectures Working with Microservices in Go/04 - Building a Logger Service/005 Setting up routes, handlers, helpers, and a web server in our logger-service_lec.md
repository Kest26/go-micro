## Настройка маршрутов, обработчиков, помощников и запуск веб-сервера в нашем логгер-сервисе

### **1. Подготовка к запуску веб-сервера**

Теперь, когда наша модель данных для логгера настроена, мы можем вернуться к файлу `main.go`, который находится в сервисе логгирования в папке `CMD API`, и внести несколько изменений.

Первое, что нужно сделать — это добавить наши модели. В блоке типа `Config` в файле `main.go` мы добавим поле `Models`, которое будет типа `data.Models`. При этом Go автоматически импортирует необходимый пакет, что удобно.

После этого, ниже в функции `main`, после отложенного вызова `defer func` для закрытия соединения, мы настроим переменную `app`. Она будет инициализироваться значением структуры `Config`, где поле `Models` будет получать значение от вызова функции `data.New`, которая требует клиента MongoDB. Этот клиент у нас уже есть, так что просто передаем его.

Теперь, когда у нас есть переменная `app`, следующим шагом будет запуск веб-сервера.

### **2. Создание функции для запуска сервера**

Чтобы упростить код, особенно с учетом того, что в дальнейшем мы будем добавлять поддержку RPC и gRPC, мы создадим отдельную функцию для запуска сервера. Это будет метод с получателем `app` (указатель на `Config`), и назовем его `serve`. Эта функция не будет принимать никаких параметров.

Функция `serve` будет просто запускать веб-сервер. Мы создаем переменную `srv` и присваиваем ей значение нового HTTP-сервера:

```go
srv := &http.Server{
    Addr:    fmt.Sprintf(":%s", webPort),
    Handler: app.routes(),
}
```

Поле `Addr` задает адрес, на котором будет слушать сервер, а `Handler` будет обрабатывать маршруты, которые мы определим позже. Для адреса мы используем форматирование строки с использованием нашей константы `webPort`.

Теперь давайте перейдем к настройке маршрутов.

### **3. Настройка маршрутов и подключение middleware**

Для маршрутизации запросов нам нужно создать новый файл `routes.go` в папке `CMD API`. В этом файле мы определим маршруты и подключим middleware. Перед этим убедимся, что необходимые пакеты установлены:

1. Пакет для маршрутизации `chi`:

    ```bash
    go get github.com/go-chi/chi/v5
    ```

2. Пакет `middleware` для работы с промежуточными обработчиками:

    ```bash
    go get github.com/go-chi/chi/middleware
    ```

3. Пакет `cors` для управления политикой кросс-доменных запросов:

    ```bash
    go get github.com/go-chi/cors
    ```

После установки пакетов мы создаем файл `routes.go` и определяем маршруты. Здесь мы можем ускорить процесс, скопировав маршруты из другого сервиса (например, `broker`) и адаптировав их под текущие нужды. После этого изменим ненужные маршруты и добавим новый маршрут для логгирования, который будет обрабатывать запросы на путь `/log` с помощью метода `app.writeLog`.

Пример кода для маршрутов:

```go
func (app *Config) routes() http.Handler {
    mux := chi.NewRouter()

    // Настройка CORS
    mux.Use(cors.Handler(cors.Options{
        AllowedOrigins:   []string{"https://*", "http://*"},
        AllowedMethods:   []string{"GET", "POST", "PUT", "DELETE", "OPTIONS"},
        AllowedHeaders:   []string{"Accept", "Authorization", "Content-Type", "X-CSRF-Token"},
        ExposedHeaders:   []string{"Link"},
        AllowCredentials: true,
        MaxAge:           300,
    }))

    // Подключение middleware
    mux.Use(middleware.Heartbeat("/ping"))

    // Определение маршрута
    mux.Post("/log", app.WriteLog)

    return mux
}
```

Теперь, когда маршруты настроены, создадим файл `handlers.go`, в котором будем обрабатывать запросы.

### **4. Обработка POST-запроса на логгирование**

В файле `handlers.go` мы определим метод `WriteLog`, который будет обрабатывать POST-запросы на путь `/log`. Метод будет принимать два параметра: `http.ResponseWriter` и `*http.Request`. Основная задача метода — получить и обработать JSON-данные, отправленные в запросе, и записать их в базу данных.

Для работы с JSON создадим структуру `JSONPayload`, которая будет описывать ожидаемое тело запроса:

```go
type JSONPayload struct {
    Name string `json:"name"`
    Data string `json:"data"`
}
```

Метод `WriteLog` сначала прочитает JSON-данные из запроса в переменную `requestPayload`, используя метод `app.readJSON`, который мы добавим в файл `helpers.go`. Затем эти данные будут использоваться для создания записи логгирования (`LogEntry`) и вставки её в базу данных:

```go
event := data.LogEntry{
    Name: requestPayload.Name,
    Data: requestPayload.Data,
}

err := app.Models.LogEntry.Insert(event)
if err != nil {
    app.errorJSON(w, err)
    return
}
```

Если вставка прошла успешно, вернем JSON-ответ с сообщением о том, что лог был успешно записан.

### **5. Вспомогательные функции для работы с JSON**

Функции для чтения JSON, записи JSON-ответа и обработки ошибок определены в файле `helpers.go`. Эти функции позволяют обрабатывать JSON-запросы и формировать корректные ответы.

-   **readJSON**: читает и проверяет JSON из запроса.
-   **writeJSON**: формирует JSON-ответ и отправляет его клиенту.
-   **errorJSON**: формирует и отправляет JSON-ответ в случае ошибки.

### **6. Запуск сервера**

Теперь, когда всё настроено, вернемся к файлу `main.go` и добавим запуск нашего веб-сервера. Для этого вызовем метод `serve` в отдельной горутине:

```go
go app.serve()
```

#### Почему запускаем сервер в горутине?

Запуск сервера в горутине позволяет основной программе продолжить выполнение других задач параллельно с работой сервера. Например, программа может выполнять мониторинг других служб, управлять соединениями или обрабатывать другие процессы. Если бы мы вызвали `serve` как обычную функцию, выполнение программы остановилось бы на этом вызове, и никаких других задач не выполнялось бы.

#### **7. Подготовка к тестированию и возможные ошибки**

После настройки сервера, в следующей лекции мы изменим строку подключения MongoDB с `mongodb://mongo` на `localhost://mongo`, чтобы протестировать приложение локально. Также обновим файл `docker-compose.yml`, чтобы запустить экземпляр MongoDB, который будет доступен как внутри Docker, так и на локальном хосте.

Мы можем столкнуться с ошибками, так как было написано много кода без промежуточных тестов. В следующей лекции мы запустим приложение и устраним любые ошибки, если они возникнут.

---
