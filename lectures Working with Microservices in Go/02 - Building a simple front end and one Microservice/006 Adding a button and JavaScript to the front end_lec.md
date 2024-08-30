## Лекция

Итак, наш сервис брокера запущен в фоновом режиме, и теперь мы хотим изменить наш фронтенд, чтобы он мог отправлять запросы к брокеру. Мы хотим убедиться, что можем отправить запрос и получить ответ в формате JSON от нашего сервиса брокера.

### Изменение шаблона фронтенда

Для начала, откроем папку с шаблонами и откроем файл `test.page.gohtml`. Это просто старый добрый JavaScript, поэтому задача не должна быть слишком сложной.

#### Добавление кнопки

Первым делом добавим кнопку. В коде, на строке 8, после горизонтальной линии добавим кнопку с идентификатором `brokerBtn`. Используем классы Bootstrap для стилизации кнопки:

```html
<a id="brokerBtn" class="btn btn-outline-secondary" href="javascript:void(0);">Test Broker</a>
```

Теперь у нас есть элемент, на который можно нажать.

#### Работа с JavaScript

Теперь перейдем к разделу с JavaScript. Первым делом получим ссылки на элементы страницы:

```javascript
let brokerBtn = document.getElementById("brokerBtn");
let output = document.getElementById("output");
let sent = document.getElementById("payload");
let received = document.getElementById("received");
```

Таким образом, у нас есть ссылки на кнопку, область вывода, и элементы для отображения отправленных и полученных данных.

#### Добавление слушателя событий

Добавим слушатель событий для нашей кнопки, чтобы отслеживать нажатия:

```javascript
brokerBtn.addEventListener("click", function() {
    const body = {
        method: 'POST',
    }

    fetch("http:\/\/localhost:8080", body)
    .then((response) => response.json())
    .then((data) => {
        sent.innerHTML = "empty post request";
        received.innerHTML = JSON.stringify(data, undefined, 4);
        if (data.error) {
            console.log(data.message);
        } else {
            output.innerHTML += `<br><strong>Response from broker service</strong>: ${data.message}`;
        }
    })
    .catch((error) => {
        output.innerHTML += "<br><br>Error: " + error;
    })
});
```

#### Объяснение кода JavaScript

1. **Создание тела запроса**: Мы создаем объект `body`, который определяет метод запроса как POST.
2. **Выполнение fetch-запроса**: Используем функцию `fetch`, чтобы отправить POST-запрос на локальный сервер по адресу `http://localhost:8080`.
3. **Обработка ответа**: После получения ответа, преобразуем его в JSON и обновляем содержимое элементов `sent` и `received` с помощью данных из ответа.
4. **Обработка ошибок**: Если произойдет ошибка, она будет выведена в область `output`.

### Запуск и тестирование

Теперь запустим наш фронтенд:

```bash
go run ./cmd/web
```

Убедитесь, что образ Docker с сервисом брокера запущен в фоновом режиме. Откроем браузер и обновим страницу. Мы также откроем консоль разработчика в браузере, чтобы видеть возможные ошибки.

#### Тестирование

1. **Нажимаем на кнопку "Test Broker"**.
2. **Проверяем результат**: Если все настроено правильно, мы увидим сообщение "empty post request" в разделе "Sent" и JSON-ответ от сервиса брокера в разделе "Received".

### Заключение

Таким образом, мы настроили фронтенд для взаимодействия с нашим сервисом брокера и протестировали его работу. Если вы видите ошибки в консоли браузера, их нужно исправить, как это было показано в лекции (например, исправление синтаксической ошибки в `innerHTML`).

---

### Полный шаблон `test.page.gohtml`

```html
{{template "base" .}}

{{define "content" }}
    <div class="container">
        <div class="row">
            <div class="col">
                <h1 class="mt-5">Test microservices</h1>
                <hr>
                <a id="brokerBtn" class="btn btn-outline-secondary" href="javascript:void(0);">Test Broker</a>

                <div id="output" class="mt-5" style="outline: 1px solid silver; padding: 2em;">
                    <span class="text-muted">Output shows here...</span>
                </div>
            </div>
        </div>
        <div class="row">
            <div class="col">
                <h4 class="mt-5">Sent</h4>
                <div class="mt-1" style="outline: 1px solid silver; padding: 2em;">
                    <pre id="payload"><span class="text-muted">Nothing sent yet...</span></pre>
                </div>
            </div>
            <div class="col">
                <h4 class="mt-5">Received</h4>
                <div class="mt-1" style="outline: 1px solid silver; padding: 2em;">
                    <pre id="received"><span class="text-muted">Nothing received yet...</span></pre>
                </div>
            </div>
        </div>
    </div>
{{end}}

{{define "js"}}
    <script>
    let brokerBtn = document.getElementById("brokerBtn");
    let output = document.getElementById("output");
    let sent = document.getElementById("payload");
    let received = document.getElementById("received");

    brokerBtn.addEventListener("click", function() {
        const body = {
            method: 'POST',
        }

        fetch("http:\/\/localhost:8080", body)
        .then((response) => response.json())
        .then((data) => {
            sent.innerHTML = "empty post request";
            received.innerHTML = JSON.stringify(data, undefined, 4);
            if (data.error) {
                console.log(data.message);
            } else {
                output.innerHTML += `<br><strong>Response from broker service</strong>: ${data.message}`;
            }
        })
        .catch((error) => {
            output.innerHTML += "<br><br>Error: " + error;
        })
    })
    </script>
{{end}}
```

Теперь вы можете видеть, как фронтенд взаимодействует с backend-сервисом, и понимать каждый шаг этого процесса.
