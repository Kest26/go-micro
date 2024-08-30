## Построение маршрутов, обработчиков и email-шаблонов

### 1. Подготовка шаблонов для email

Перед тем как начать отправлять email-сообщения, нам нужно создать несколько шаблонов, которые будут использоваться для построения содержимого писем. В файле `mailer.go`, который находится в папке `cmd/api` сервиса почты, ожидается наличие двух шаблонов: `mail.html.gohtml` для HTML-сообщений и `mail.plain.gohtml` для простого текста.

Для этого на корневом уровне проекта создадим новую папку `templates` и в ней два файла: `mail.html.gohtml` и `mail.plain.gohtml`. Названия этих файлов должны соответствовать именам, которые указаны в коде файла `mailer.go`. Важно, чтобы названия совпадали, так как именно на них ссылается логика работы программы.

#### Создание шаблона для простого текста

Начнем с создания шаблона для простого текста (`mail.plain.gohtml`). Этот шаблон очень простой. Внутри него мы определяем блок `body`, который содержит только одну переменную `.message`. Эта переменная отвечает за вывод содержимого сообщения, которое будет передаваться при отправке письма.

Пример содержимого шаблона:

```html
{{define "body"}} {{.message}} {{end}}
```

Такое определение `body` соответствует тому, что мы видим в коде в файле `mailer.go`, где происходит выполнение шаблона, и результат помещается в переменную `tpl`. Важным является то, что `.message` — это ключевой элемент, который хранит контент сообщения. Он берется из данных, переданных в шаблон.

#### Создание HTML-шаблона

Теперь перейдем к созданию HTML-версии шаблона (`mail.html.gohtml`). Этот шаблон будет более сложным. Внутри него мы также определяем блок `body` и добавляем базовую структуру HTML-документа.

Пример содержимого шаблона:

```html
{{define "body"}}
<!DOCTYPE html>
<html lang="en">
    <head>
        <meta name="viewport" content="width=device-width" />
        <meta http-equiv="Content-Type" content="text/html; charset=UTF-8" />
        <title></title>
    </head>
    <body>
        <p>{{.message}}</p>
    </body>
</html>
{{end}}
```

Здесь мы задаем структуру HTML-документа: doctype, тег `html` с атрибутом `lang="en"`, мета-теги для настройки отображения и кодировки, а также пустой тег `title`. Основное содержимое письма размещается в блоке `<p>` через переменную `.message`. Это позволяет вставлять текст сообщения в HTML-контекст.

### 2. Добавление маршрута и обработчика

Теперь, когда шаблоны готовы, нужно добавить маршрут, который будет обрабатывать запросы на отправку email. Для этого в файле `routes.go` добавим новый маршрут, который будет использовать метод POST и указывать на обработчик, которого пока еще нет.

Код маршрута:

```go
mux.Post("/send", app.SendMail)
```

Этот маршрут будет направлять запросы по пути `/send` на обработчик `SendMail`. Обработчик пока не реализован, поэтому давайте его создадим.

### 3. Реализация обработчика отправки email

Создадим новый файл `handlers.go` в папке `cmd/api`. Этот файл будет содержать обработчик для отправки email. Обработчик реализован как метод `SendMail`, который принимает в качестве параметров `http.ResponseWriter` и `*http.Request`.

Пример кода:

```go
package main

import "net/http"

func (app *Config) SendMail(w http.ResponseWriter, r *http.Request) {
	type mailMessage struct {
		From    string `json:"from"`
		To      string `json:"to"`
		Subject string `json:"subject"`
		Message string `json:"message"`
	}

	var requestPayload mailMessage

	err := app.readJSON(w, r, &requestPayload)
	if err != nil {
		app.errorJSON(w, err)
		return
	}

	msg := Message{
		From:    requestPayload.From,
		To:      requestPayload.To,
		Subject: requestPayload.Subject,
		Data:    requestPayload.Message,
	}

	err = app.Mailer.SendSMTPMessage(msg)
	if err != nil {
		app.errorJSON(w, err)
		return
	}

	payload := jsonResponse{
		Error:   false,
		Message: "sent to " + requestPayload.To,
	}

	app.writeJSON(w, http.StatusAccepted, payload)
}
```

Здесь мы сначала создаем структуру `mailMessage`, которая описывает поля ожидаемого JSON-запроса. Затем считываем данные из запроса и проверяем на наличие ошибок. Если данные валидны, формируем сообщение типа `Message` и передаем его на отправку через `app.Mailer.SendSMTPMessage`.

Если отправка проходит успешно, возвращаем клиенту JSON-ответ с подтверждением.

### 4. Настройка mailer в конфигурации

Для отправки email нам нужно настроить mailer в конфигурации. Это делается в файле `main.go`. Добавим новое поле `Mailer` в структуру `Config` и реализуем функцию `createMail`, которая будет возвращать объект типа `Mail` с настроенными параметрами.

Пример кода:

```go
package main

import (
	"fmt"
	"log"
	"net/http"
	"os"
	"strconv"
)

type Config struct {
	Mailer Mail
}

const webPort = "80"

func main() {
	app := Config{
		Mailer: createMail(),
	}

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

func createMail() Mail {
	port, _ := strconv.Atoi(os.Getenv("MAIL_PORT"))
	m := Mail{
		Domain:      os.Getenv("MAIL_DOMAIN"),
		Host:        os.Getenv("MAIL_HOST"),
		Port:        port,
		Username:    os.Getenv("MAIL_USERNAME"),
		Password:    os.Getenv("MAIL_PASSWORD"),
		Encryption:  os.Getenv("MAIL_ENCRYPTION"),
		FromName:    os.Getenv("FROM_NAME"),
		FromAddress: os.Getenv("FROM_ADDRESS"),
	}

	return m
}
```

Функция `createMail` считывает необходимые параметры из переменных окружения, преобразует их в нужные типы и создает объект `Mail`, который затем используется в приложении для отправки email.

### Заключение

Теперь, когда все части системы готовы, мы можем построить Docker-образ и добавить его в `docker-compose.yml`. После этого можно тестировать отправку email и при необходимости добавить логику безопасности перед выходом в продакшн.
