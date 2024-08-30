## Обновление Brocker для работы со стандартным форматом JSON и подключение к нашему сервису Authentication.

#### **1. Введение: Настройка аутентификационного микросервиса**

Мы создали первую версию аутентификационного микросервиса, который уже работает в Docker. Также у нас есть база данных с необходимой информацией. Следующий шаг — модифицировать брокерное приложение, чтобы оно могло принимать запросы от фронтенда, пересылать их в аутентификационный микросервис, получать ответ и отправлять его обратно пользователю. Чтобы это сделать, нам нужно настроить новый маршрут в брокер-сервисе и создать соответствующий обработчик запросов.

#### **2. Настройка маршрута и создание обработчика в брокер-сервисе**

Для начала, мы откроем файл `routes.go` в брокер-сервисе. В нем мы добавим новый маршрут, который будет перенаправлять запросы на новый обработчик, который пока еще не существует. Этот маршрут будет обрабатывать все запросы, независимо от того, с каким микросервисом мы работаем. Мы определяем его как `mux.Post("/handle", app.HandleSubmission)`.

Затем переходим к созданию обработчика в файле `handlers.go`. Мы создадим функцию-обработчик `HandleSubmission`, которая будет принимать входящие запросы, обрабатывать их и передавать в соответствующий микросервис.

#### **3. Определение структуры данных: Типы и формат JSON**

Поскольку наш обработчик будет принимать различные запросы, важно иметь стандартизированный формат данных. Мы создадим структуру `RequestPayload`, которая будет представлять собой JSON-объект, передаваемый в брокер-сервис. Эта структура будет содержать поле `Action` (действие), определяющее, какое действие требуется выполнить, и соответствующие данные для этого действия, например, данные для аутентификации (`AuthPayload`).

```go
type RequestPayload struct {
    Action string      `json:"action"`
    Auth   AuthPayload `json:"auth,omitempty"`
}
```

#### **4. Обработка запросов: Разбор JSON и выбор действия**

Внутри функции `HandleSubmission` мы будем разбирать входящий JSON и выбирать действие в зависимости от поля `Action`. Используя конструкцию `switch`, мы определяем, какое действие необходимо выполнить (например, `auth` для аутентификации).

```go
switch requestPayload.Action {
    case "auth":
        app.authenticate(w, requestPayload.Auth)
    default:
        app.errorJSON(w, errors.New("unknown action"))
}
```

#### **5. Реализация логики аутентификации**

Для выполнения аутентификации мы создаем отдельную функцию `authenticate`, которая будет отправлять запрос на аутентификационный микросервис. В этой функции мы формируем JSON из полученных данных (`AuthPayload`), создаем HTTP-запрос, отправляем его в аутентификационный микросервис и обрабатываем ответ.

```go
func (app *Config) authenticate(w http.ResponseWriter, a AuthPayload) {
    jsonData, _ := json.MarshalIndent(a, "", "\t")

    request, err := http.NewRequest("POST", "http://authentication-service/authenticate", bytes.NewBuffer(jsonData))
    if err != nil {
        app.errorJSON(w, err)
        return
    }

    client := &http.Client{}
    response, err := client.Do(request)
    if err != nil {
        app.errorJSON(w, err)
        return
    }
    defer response.Body.Close()

    if response.StatusCode == http.StatusUnauthorized {
        app.errorJSON(w, errors.New("invalid credentials"))
        return
    } else if response.StatusCode != http.StatusAccepted {
        app.errorJSON(w, errors.New("error calling auth service"))
        return
    }

    var jsonFromService jsonResponse
    err = json.NewDecoder(response.Body).Decode(&jsonFromService)
    if err != nil {
        app.errorJSON(w, err)
        return
    }

    if jsonFromService.Error {
        app.errorJSON(w, err, http.StatusUnauthorized)
        return
    }

    var payload jsonResponse
    payload.Error = false
    payload.Message = "Authenticated!"
    payload.Data = jsonFromService.Data

    app.writeJSON(w, http.StatusAccepted, payload)
}
```

Функция `authenticate` используется для аутентификации пользователя. Она отправляет данные пользователя (email и пароль) к микросервису, который проверяет эти данные и возвращает результат — успешная или неуспешная аутентификация. Рассмотрим шаги работы функции:

### Шаги выполнения функции

1. **Создание JSON с данными пользователя:**

    - Функция получает структуру `AuthPayload`, которая содержит email и пароль пользователя.
    - Эти данные преобразуются в формат JSON, чтобы их можно было отправить в запросе.

2. **Отправка запроса к микросервису:**

    - Функция создает HTTP-запрос типа POST, который будет отправлен к микросервису аутентификации. В теле запроса содержится созданный ранее JSON.
    - URL для отправки запроса — это адрес микросервиса аутентификации.

3. **Получение и обработка ответа:**

    - После отправки запроса функция ожидает ответа от микросервиса.
    - Если ответ указывает на неуспешную аутентификацию (например, неправильный пароль), функция возвращает ошибку.
    - Если статус ответа не соответствует успешной аутентификации, функция также возвращает ошибку.

4. **Обработка успешного ответа:**
    - Если аутентификация прошла успешно, JSON-ответ от микросервиса декодируется и проверяется.
    - Если JSON содержит ошибку, функция снова возвращает ошибку.
    - Если все в порядке, функция формирует успешный ответ с подтверждением аутентификации и данными пользователя и отправляет его обратно пользователю.

#### **6. Завершение: Тестирование интеграции и настройка фронтенда**

После того как мы реализовали функцию аутентификации, мы можем перейти к тестированию. Необходимо отправить запрос с фронтенда на маршрут `/handle` брокер-сервиса с соответствующим JSON-пейлоадом. Брокер вызовет аутентификационный микросервис, получит ответ и отправит его обратно клиенту.

В следующей лекции мы сосредоточимся на реализации взаимодействия с фронтендом.

---

#### **brocker-service/routes.go**

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

#### **brocker-service/handlers.go**

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
}

type AuthPayload struct {
	Email    string `json:"email"`
	Password string `json:"password"`
}

func (app *Config) Broker(w http.ResponseWriter, r *http.Request) {
	payload := jsonResponse{
		Error:   false,
		Message: "Hit the broker",
	}

	_ = app.writeJSON(w, http.StatusOK, payload)
}

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
	default:
		app.errorJSON(w, errors.New("unknown action"))

	}
}

func (app *Config) authenticate(w http.ResponseWriter, a AuthPayload) {
	jsonData, _ := json.MarshalIndent(a, "", "\t")

	request, err := http.NewRequest("POST", "http://authentication-service/authenticate", bytes.NewBuffer(jsonData))
	if err != nil {
		app.errorJSON(w, err)
		return
	}

	client := &http.Client{}
	response, err := client.Do(request)
	if err != nil {
		app.errorJSON(w, err)
		return
	}
	defer response.Body.Close()

	if response.StatusCode == http.StatusUnauthorized {
		app.errorJSON(w, errors.New("invalid credentials"))
		return
	} else if response.StatusCode != http.StatusAccepted {
		app.errorJSON(w, errors.New("error calling auth service"))
		return
	}

	var jsonFromService jsonResponse

	err = json.NewDecoder(response.Body).Decode(&jsonFromService)
	if err != nil {
		app.errorJSON(w, err)
		return
	}

	if jsonFromService.Error {
		app.errorJSON(w, err, http.StatusUnauthorized)
		return
	}

	var payload jsonResponse

	payload.Error = false
	payload.Message = "Authenticated!"
	payload.Data = jsonFromService.Data

	app.writeJSON(w, http.StatusAccepted, payload)

}
```
