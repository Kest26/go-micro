### Тестирование микросервисов аутентификации и брокера с использованием JavaScript на фронтенде

#### 1. Подготовка фронтенда: Добавление кнопки для тестирования микросервиса аутентификации

Мы собираемся протестировать микросервис аутентификации через фронтенд. Начнем с добавления кнопки на странице, которая будет отправлять запросы к этому микросервису. Откройте файл `test.page.gohtml` и найдите место, где уже определена кнопка для тестирования брокера. Дублируем эту строку и изменяем идентификатор и текст:

```html
<a id="authBrokerBtn" class="btn btn-outline-secondary" href="javascript:void(0);">Test Auth</a>
```

Эта кнопка будет отображаться на веб-странице и использоваться для отправки запроса на аутентификацию.

#### 2. Создание обработчика событий для кнопки аутентификации

Теперь добавим JavaScript-код, который будет выполняться при нажатии на эту кнопку. Код будет отправлять HTTP-запрос в микросервис аутентификации и обрабатывать ответ.

```javascript
let authBrokerBtn = document.getElementById("authBrokerBtn");
let output = document.getElementById("output");
let sent = document.getElementById("payload");
let received = document.getElementById("received");

authBrokerBtn.addEventListener("click", function() {

    // Формируем тело запроса
    const payload = {
        action: "auth",  // Указываем действие, которое должно быть выполнено на сервере
        auth: {  // Данные для аутентификации
            email: "admin@example.com",
            password: "verysecret",
        }
    }

    // Создаем заголовки для запроса
    const headers = new Headers();
    headers.append("Content-Type", "application/json");

    // Формируем параметры запроса
    const body = {
        method: 'POST',  // Метод запроса POST
        body: JSON.stringify(payload),  // Преобразуем данные запроса в JSON-строку
        headers: headers,  // Применяем заголовки
    }

    // Отправляем запрос с помощью fetch API
    fetch("http://localhost:8080/handle", body)
        .then((response) => response.json())  // Преобразуем ответ в JSON
        .then((data) => {
            sent.innerHTML = JSON.stringify(payload, undefined, 4);  // Отображаем отправленные данные
            received.innerHTML = JSON.stringify(data, undefined, 4);  // Отображаем полученные данные

            // Если есть ошибка, отображаем её
            if (data.error) {
                output.innerHTML += `<br><strong>Error:</strong> ${data.message}`;
            } else {
                output.innerHTML += `<br><strong>Response from broker service</strong>: ${data.message}`;
            }
        })
        .catch((error) => {
            output.innerHTML += "<br><br>Error: " + error;  // Обработка ошибок запроса
        })
})
```

#### 3. Тестирование и отладка

Когда код написан, запускаем проект и проверяем работу микросервисов. Сначала необходимо убедиться, что контейнеры Docker запущены:

```bash
make up_build  # Собираем Docker-образы
make start  # Запускаем фронтенд
```

После этого открываем браузер и проверяем работу микросервиса. В случае успешного выполнения запроса на странице отобразится ответ от сервиса. Если возникнет ошибка, она будет отображена в блоке вывода.

