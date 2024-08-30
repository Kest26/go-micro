## Настройка заглушки микросервиса для отправки почты

### 1. Введение и создание структуры проекта

Настало время начать разработку микросервиса для работы с почтой. Этот процесс должен быть вам знаком. Первым шагом мы создадим новую папку для сервиса.

Перейдите в файловый менеджер и в директории `Go Micro` создайте новую папку с именем `Mail-Service`. После этого вернитесь в вашу IDE и добавьте эту папку в рабочее пространство: выберите в меню "File" опцию "Add Folder to Workspace" и найдите созданную папку.

Теперь переключитесь в эту папку в терминале: поднимитесь на один уровень вверх и перейдите в директорию `Mail-Service`. Инициализируйте новый модуль с помощью команды:

```sh
go mod init Mailer-Service
```

Не забудьте правильно указать имя сервиса.

### 2. Создание структуры проекта и основных файлов

Внутри папки `Mail-Service` создайте новую папку `CMD`, а в ней — папку `API`, так как этот микросервис не будет иметь веб-интерфейса.

Теперь создайте файл `main.go` в папке `API`:

```go
package main

import (
	"fmt"
	"log"
	"net/http"
)

type Config struct {
}

const webPort = "80"

func main() {
	app := Config{}

	log.Println("Starting mail service on port", webPort)

	srv := &http.Server{
		Addr:    fmt.Sprintf(":%s", webPort),
		Handler: app.routes(),
	}

	err := srv.ListenAndServe()
	if err != nil {
		log.Panic(err)
	}
}
```

В этом файле мы объявляем структуру `Config`, которая пока пустая, но позже мы её заполним. Также мы определяем порт, на котором будет работать сервис (`80`) и создаем сервер с использованием маршрутов.

### 3. Копирование вспомогательных функций

Мы будем работать с JSON в этом сервисе, поэтому давайте упростим задачу, скопировав необходимые функции из других сервисов. Перейдите в сервис брокера и откройте файл `helpers.go` в папке `CMD/API`. Скопируйте его содержимое и вставьте в новый файл `helpers.go`, который нужно создать в папке `API` вашего нового сервиса.

Код будет следующим:

```go
package main

import (
	"encoding/json"
	"errors"
	"io"
	"net/http"
)

type jsonResponse struct {
	Error   bool   `json:"error"`
	Message string `json:"message"`
	Data    any    `json:"data,omitempty"`
}

func (app *Config) readJSON(w http.ResponseWriter, r *http.Request, data any) error {
	maxBytes := 1048576 // one megabyte

	r.Body = http.MaxBytesReader(w, r.Body, int64(maxBytes))

	dec := json.NewDecoder(r.Body)
	err := dec.Decode(data)
	if err != nil {
		return err
	}

	err = dec.Decode(&struct{}{})
	if err != io.EOF {
		return errors.New("body must have only a single JSON value")
	}

	return nil
}

func (app *Config) writeJSON(w http.ResponseWriter, status int, data any, headers ...http.Header) error {
	out, err := json.Marshal(data)
	if err != nil {
		return err
	}

	if len(headers) > 0 {
		for key, value := range headers[0] {
			w.Header()[key] = value
		}
	}

	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	_, err = w.Write(out)
	if err != nil {
		return err
	}

	return nil
}

func (app *Config) errorJSON(w http.ResponseWriter, err error, status ...int) error {
	statusCode := http.StatusBadRequest

	if len(status) > 0 {
		statusCode = status[0]
	}

	var payload jsonResponse
	payload.Error = true
	payload.Message = err.Error()

	return app.writeJSON(w, statusCode, payload)
}
```

Эти функции позволят нам обрабатывать JSON-запросы и формировать ответы в JSON-формате.

### 4. Настройка маршрутов

Следующим шагом мы настроим маршруты. Создайте новый файл `routes.go` в папке `API` и снова упростим задачу, скопировав код из сервиса брокера.

Код будет таким:

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

	// specify who is allowed to connect
	mux.Use(cors.Handler(cors.Options{
		AllowedOrigins:   []string{"https://*", "http://*"},
		AllowedMethods:   []string{"GET", "POST", "PUT", "DELETE", "OPTIONS"},
		AllowedHeaders:   []string{"Accept", "Authorization", "Content-Type", "X-CSRF-Token"},
		ExposedHeaders:   []string{"Link"},
		AllowCredentials: true,
		MaxAge:           300,
	}))

	mux.Use(middleware.Heartbeat("/ping"))

	return mux
}
```

Этот код настраивает CORS и добавляет маршрут для проверки доступности сервиса (`/ping`).

### 5. Установка зависимостей

Для работы с маршрутами нам нужно установить несколько пакетов. Выполните следующие команды:

```sh
go get github.com/go-chi/chi/v5
go get github.com/go-chi/chi/v5/middleware
go get github.com/go-chi/cors
```

Эти пакеты добавляют поддержку маршрутизации, middleware и настройки CORS.

### 6. Запуск сервера

Теперь у нас есть основной файл `main.go`, настроенные маршруты и вспомогательные функции. Мы можем запустить сервер, который будет слушать запросы на порту 80. В данный момент маршруты пока не делают ничего полезного, кроме проверки связи, но это изменится в дальнейшем.

### 7. Следующие шаги

На следующем этапе мы добавим функционал для обработки запросов на отправку писем. Мы создадим логики для обработки запросов, генерации и отправки писем, и настроим сам почтовый сервис.


