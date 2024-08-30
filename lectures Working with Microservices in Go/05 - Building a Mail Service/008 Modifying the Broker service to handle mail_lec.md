## Модификация брокер-сервиса для обработки почтовых запросов

Прежде чем мы убедимся, что наш почтовый микросервис работает так, как ожидается, нужно внести некоторые изменения в брокер. Мы модифицируем брокер, чтобы он мог принимать JSON-запросы от фронтенда, обрабатывать их, перенаправлять на почтовый сервис и возвращать соответствующий ответ. 

**Важно**: В реальном приложении такой подход не применим. Обычно микросервисы общаются между собой напрямую и не принимают запросы от фронтенда, особенно в случае с почтовым сервисом, чтобы избежать проблем, например, с отправкой спама. Однако, в рамках разработки мы можем позволить себе эту схему, чтобы убедиться, что все работает корректно.

### 1. Обновление маршрутов в `routes.go`

Для начала перейдем к файлу `routes.go`, который управляет маршрутами нашего приложения. Мы добавим новый маршрут, который будет обрабатывать запросы на отправку почты:

```go
package main

import (
	"net/http"

	"github.com/go-chi/chi/middleware"
	"github.com/go-chi/chi/v5"
	"github.com/go-chi/cors"
)

func (app *Config) routes() http.Handler {
	mux := chi.NewRouter()

	// Разрешаем подключения с определенных источников
	mux.Use(cors.Handler(cors.Options{
		AllowedOrigins:   []string{"https://*", "http://*"},
		AllowedMethods:   []string{"GET", "POST", "PUT", "DELETE", "OPTIONS"},
		AllowedHeaders:   []string{"Accept", "Authorization", "Content-Type", "X-CSRF-Token"},
		ExposedHeaders:   []string{"Link"},
		AllowCredentials: true,
		MaxAge:           300,
	}))

	mux.Use(middleware.Heartbeat("/ping"))

	mux.Post("/", app.Broker)
	mux.Post("/handle", app.HandleSubmission)

	return mux
}
```

Здесь добавлен новый маршрут `"/handle"`, который обрабатывает запросы на отправку почты. В этом маршруте вызывается метод `HandleSubmission`, который отвечает за анализ поступившего JSON и выполнение соответствующего действия.

### 2. Описание структуры данных для почтовых запросов

Теперь перейдем к файлу `handlers.go`. В этом файле мы расширяем структуру `RequestPayload`, добавив в нее поддержку почтовых запросов:

```go
package main

import (
	"bytes"
	"encoding/json"
	"errors"
	"net/http"
)

type RequestPayload struct {
	Action string      `json:"action"`
	Auth   AuthPayload `json:"auth,omitempty"`
	Log    LogPayload  `json:"log,omitempty"`
	Mail   MailPayload `json:"mail,omitempty"`
}

type MailPayload struct {
	From    string `json:"from"`
	To      string `json:"to"`
	Subject string `json:"subject"`
	Message string `json:"message"`
}
```

Добавление нового типа `MailPayload` позволяет описать структуру данных, которая будет использоваться при отправке почтовых запросов. Эта структура включает такие поля, как отправитель (`From`), получатель (`To`), тема письма (`Subject`) и само сообщение (`Message`). 

### 3. Обработка запросов на отправку почты в `HandleSubmission`

Теперь займемся функцией `HandleSubmission`, которая обрабатывает входящие запросы. В нее добавляем новый кейс для действия `mail`, который будет вызывать функцию `sendMail`:

```go
func (app *Config) HandleSubmission(w http.ResponseWriter, r *http.Request) {
	var requestPayload RequestPayload

	// Чтение JSON-запроса
	err := app.readJSON(w, r, &requestPayload)
	if err != nil {
		app.errorJSON(w, err)
		return
	}

	// Выполнение действия в зависимости от значения "action"
	switch requestPayload.Action {
	case "auth":
		app.authenticate(w, requestPayload.Auth)
	case "log":
		app.logItem(w, requestPayload.Log)
	case "mail":
		app.sendMail(w, requestPayload.Mail)
	default:
		app.errorJSON(w, errors.New("unknown action"))
	}
}
```

Функция сначала считывает входящий JSON-запрос и определяет, какое действие необходимо выполнить на основании значения поля `Action`. Если действие — отправка почты (`"mail"`), вызывается функция `sendMail`, которая и отвечает за выполнение этого действия.

### 4. Реализация функции отправки почты

Теперь рассмотрим функцию `sendMail`, которая отправляет почту через почтовый микросервис:

```go
func (app *Config) sendMail(w http.ResponseWriter, msg MailPayload) {
	// Преобразуем сообщение в формат JSON
	jsonData, _ := json.MarshalIndent(msg, "", "\t")

	// Определяем URL почтового сервиса
	mailServiceURL := "http://mail-service/send"

	// Создаем HTTP-запрос
	request, err := http.NewRequest("POST", mailServiceURL, bytes.NewBuffer(jsonData))
	if err != nil {
		app.errorJSON(w, err)
		return
	}

	// Устанавливаем заголовок Content-Type
	request.Header.Set("Content-Type", "application/json")

	// Инициализируем HTTP-клиент и выполняем запрос
	client := &http.Client{}
	response, err := client.Do(request)
	if err != nil {
		app.errorJSON(w, err)
		return
	}
	defer response.Body.Close()

	// Проверяем статус-код ответа
	if response.StatusCode != http.StatusAccepted {
		app.errorJSON(w, errors.New("error calling mail service"))
		return
	}

	// Формируем и отправляем ответ обратно
	var payload jsonResponse
	payload.Error = false
	payload.Message = "Message sent to " + msg.To

	app.writeJSON(w, http.StatusAccepted, payload)
}
```

Функция `sendMail` берет данные о письме, которые она получает в виде структуры `MailPayload`, и преобразует их в JSON. Затем она создает HTTP-запрос типа POST на URL почтового сервиса и отправляет этот запрос с JSON-данными. 

Важно понимать, почему мы используем функцию `json.MarshalIndent`. Она помогает отформатировать JSON с отступами, что делает его удобным для чтения на этапе разработки. Однако в продакшене этот подход можно заменить на `json.Marshal`, чтобы избежать лишнего форматирования и сократить размер данных.

### 5. Взаимодействие с почтовым сервисом

После отправки запроса к почтовому сервису мы проверяем статус-код ответа. Ожидается, что почтовый сервис вернет код `http.StatusAccepted`, если письмо было успешно отправлено. В случае ошибки, мы возвращаем сообщение об ошибке.

Если все прошло успешно, мы формируем JSON-ответ и отправляем его обратно клиенту, указывая, что письмо было отправлено.

### 6. Следующие шаги

Теперь, когда основная логика обработки почтовых запросов реализована, следующим шагом будет обновление фронтенда. Нужно будет добавить кнопку и написать JavaScript-код, чтобы протестировать процесс отправки почты. Мы займемся этим в следующем уроке и проверим, что все работает правильно.

---

Эти изменения обеспечивают связь между фронтендом, брокером и почтовым микросервисом, что позволит нам протестировать систему в полной мере.
