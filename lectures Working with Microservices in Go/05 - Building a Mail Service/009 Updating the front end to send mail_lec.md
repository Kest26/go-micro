## Обновление фронтенда для отправки писем

Мы настроили брокерный сервис и сервис для отправки писем. Теперь настало время протестировать их в работе. Для этого перейдём к фронтенду и внесём необходимые изменения.

### 1. Добавление кнопки для отправки писем на странице тестирования

Откроем файл `test.page.gohtml`, который находится в папке `cmd/web/templates/`. Нам нужно добавить кнопку для отправки письма. Мы можем дублировать одну из существующих кнопок и изменить её параметры.

```html
<a id="brokerBtn" class="btn btn-outline-secondary" href="javascript:void(0);"
    >Test Broker</a
>
<a
    id="authBrokerBtn"
    class="btn btn-outline-secondary"
    href="javascript:void(0);"
    >Test Auth</a
>
<a id="logBtn" class="btn btn-outline-secondary" href="javascript:void(0);"
    >Test Log</a
>
<a id="mailBtn" class="btn btn-outline-secondary" href="javascript:void(0);"
    >Test Mail</a
>
```

Теперь у нас есть кнопка `Test Mail` с идентификатором `mailBtn`.

### 2. Добавление JavaScript для обработки клика по кнопке

Перейдём к разделу JavaScript в этом же файле и добавим обработчик события для нашей новой кнопки. Мы можем дублировать существующий код и внести в него изменения.

**Изменённый код для кнопки `Test Mail`:**

```javascript
let mailBtn = document.getElementById("mailBtn");

mailBtn.addEventListener("click", function () {
    const payload = {
        action: "mail",
        mail: {
            from: "me@example.com",
            to: "you@there.com",
            subject: "Test email",
            message: "Hello world!",
        },
    };

    const headers = new Headers();
    headers.append("Content-Type", "application/json");

    const body = {
        method: "POST",
        body: JSON.stringify(payload),
        headers: headers,
    };

    fetch("http://localhost:8080/handle", body)
        .then((response) => response.json())
        .then((data) => {
            sent.innerHTML = JSON.stringify(payload, undefined, 4);
            received.innerHTML = JSON.stringify(data, undefined, 4);
            if (data.error) {
                output.innerHTML += `<br><strong>Error:</strong> ${data.message}`;
            } else {
                output.innerHTML += `<br><strong>Response from broker service</strong>: ${data.message}`;
            }
        })
        .catch((error) => {
            output.innerHTML += "<br><br>Error: " + error;
        });
});
```

Этот код выполняет следующие задачи:

1. Создаёт JSON-пакет с данными для отправки письма.
2. Настраивает заголовки и тело запроса.
3. Отправляет запрос на наш брокерный сервис по адресу `http://localhost:8080/handle`.
4. Отображает результат в соответствующих полях на странице.

### 3. Исправление ошибок в брокерном сервисе

Теперь давайте проверим, что наш код в брокерном сервисе правильно обрабатывает запросы на отправку почты. Для этого откроем файл `handlers.go` в брокерном сервисе и убедимся, что обработчик `sendMail` настроен правильно.

**Код обработчика `sendMail`:**

```go
func (app *Config) sendMail(w http.ResponseWriter, msg MailPayload) {
    jsonData, _ := json.MarshalIndent(msg, "", "\t")

    // Адрес сервиса для отправки почты
    mailServiceURL := "http://mailer-service/send"

    // Создание запроса к сервису
    request, err := http.NewRequest("POST", mailServiceURL, bytes.NewBuffer(jsonData))
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

    // Проверка статуса ответа
    if response.StatusCode != http.StatusAccepted {
        app.errorJSON(w, errors.New("error calling mail service"))
        return
    }

    // Формирование ответа для фронтенда
    var payload jsonResponse
    payload.Error = false
    payload.Message = "Message sent to " + msg.To

    app.writeJSON(w, http.StatusAccepted, payload)
}
```

Этот код выполняет следующие задачи:

1. Формирует JSON-пакет с данными для отправки на почтовый сервис.
2. Отправляет запрос на почтовый сервис по адресу `http://mailer-service/send`.
3. Обрабатывает ответ от почтового сервиса и отправляет результат на фронтенд.

### 4. Исправление имени сервиса в Docker Compose

Мы столкнулись с ошибкой при обращении к почтовому сервису, так как в `Docker Compose` указано правильное имя для сервиса ``mailer-service:

```yaml
services:
    mailer-service: ...
```

А в `brocker-service` в функции `sendMail` для вызова сервиса мы испольуем имя `mail-service`. Вот и ошибка, здесь мы написали имя `mail-service`. Давайте исправим.

```go
func (app *Config) sendMail(w http.ResponseWriter, msg MailPayload) {
	jsonData, _ := json.MarshalIndent(msg, "", "\t")

	// call the mail service
	mailServiceURL := "http://mailer-service/send"

    ...
}

```

### 5. Исправление в `mail-service.dockerfile`

Наш почтовый сервер не мог найти шаблоны для отправки писем. Это мы исправили, добавив в `mail-service.dockerfile` команду `COPY templates /templates`, которая копирует папку с шаблонами писем из локального окружения в контейнер Docker. Благодаря этому почтовый сервис смог правильно найти и использовать эти шаблоны для отправки писем.

```dockerfile
FROM alpine:latest

RUN mkdir /app

COPY mailerApp /app
COPY templates /templates

CMD [ "/app/mailerApp"]

```

#### Объяснение:

-   **FROM alpine:latest** — используем базовый образ `alpine`, который является лёгким Linux-дистрибутивом.
-   **RUN mkdir /app** — создаём директорию `/app` внутри контейнера для размещения файлов приложения.
-   **COPY mailerApp /app** — копируем исполняемый файл приложения (например, скомпилированный бинарник Go) в директорию `/app` контейнера.
-   **COPY templates /templates** — копируем папку с шаблонами писем в контейнер. Это важно для работы почтового сервиса, поскольку шаблоны используются для формирования тела писем.
-   **CMD [ "/app/mailerApp"]** — запускаем приложение при старте контейнера.

Таким образом, добавление команды `COPY templates /templates` устранило проблему, связанную с отсутствием шаблонов писем в контейнере, и позволило почтовому сервису корректно работать.

#### 6. Тестирование изменений

Теперь мы готовы протестировать изменения. Для этого выполните следующие команды:

```bash
make down
make up_build
make start
```

Перейдите в браузер и откройте страницу с тестами. Нажмите на кнопку `Test Mail` и проверьте, получено ли письмо в `MailHog` (обычно доступен по адресу `localhost:8025`).

Если всё настроено правильно, вы увидите сообщение "Message sent to you@there.com", а в `MailHog` будет отображаться письмо с текстом "Hello world!".

### Заключение

Мы успешно добавили функциональность отправки писем на фронтенд, а также убедились, что брокерный и почтовый сервисы работают корректно. Не забывайте проверять правильность конфигурации Docker Compose, dockerfile и структуры ваших файлов при возникновении ошибок.
