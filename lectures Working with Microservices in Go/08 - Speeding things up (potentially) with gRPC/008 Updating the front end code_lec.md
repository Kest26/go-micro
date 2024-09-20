### Обновление фронтенда для работы с gRPC

#### Введение
В этой лекции мы обновим код фронтенда, чтобы протестировать наше подключение к gRPC-сервису. Мы создадим новый элемент интерфейса — кнопку для отправки запроса на сервер через gRPC. Для этого внесём изменения в HTML-шаблон и добавим JavaScript-код для обработки событий нажатия кнопки и отправки запроса.

#### Шаг 1: Добавление новой кнопки
Первым делом мы откроем HTML-шаблон `test.page.gohtml`, расположенный по пути `front-end/cmd/web/templates`. В этом файле уже имеются кнопки для тестирования различных микросервисов (например, брокера и логгирования). Мы создадим новую кнопку для тестирования gRPC.

Открываем файл и добавляем новый элемент кнопки:

```html
<a id="logGBtn" class="btn btn-outline-secondary" href="javascript:void(0);">Test gRPC log</a>
```

Эта кнопка будет иметь идентификатор `logGBtn`, чтобы позже можно было ссылаться на неё в скриптах. Название кнопки — "Test gRPC log".

**Пример из кода**:
В шаблоне мы видим несколько кнопок, одна из которых теперь выглядит так:

```html
<a id="logGBtn" class="btn btn-outline-secondary" href="javascript:void(0);">Test gRPC log</a>
```

#### Шаг 2: Изменение JavaScript-кода
Теперь переходим к секции JavaScript в том же файле. В этой секции уже есть обработчики событий для других кнопок, и нам нужно добавить обработчик для новой кнопки `logGBtn`. Для этого скопируем существующий код, отвечающий за обработку нажатий на другие кнопки, и изменим его.

Добавляем новый обработчик для кнопки `logGBtn`:

```javascript
logGBtn.addEventListener("click", function() {
    const payload = {
        action: "log",
        log: {
            name: "event",
            data: "Some kind of gRPC data",
        }
    }

    const headers = new Headers();
    headers.append("Content-Type", "application/json");

    const body = {
        method: "POST",
        body: JSON.stringify(payload),
        headers: headers,
    }

    fetch("http://localhost:8080/log-grpc", body)
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
    })
})
```

#### Объяснение:

1. **Добавление слушателя событий**:
   Мы создаём слушатель для кнопки `logGBtn`, который будет реагировать на её нажатие. Когда пользователь нажимает на кнопку, вызывается функция, внутри которой формируется запрос к серверу через gRPC.

2. **Формирование полезной нагрузки (payload)**:
   Мы создаём объект `payload`, который содержит данные для отправки на сервер. В данном случае это объект с полями `name` и `data`, где `name` — это название события, а `data` — сами данные (строка `"Some kind of gRPC data"`). Эти данные будут отправлены серверу для записи в лог через gRPC.

3. **Настройка заголовков**:
   Мы добавляем заголовок, указывающий тип содержимого запроса как `application/json`.

4. **Отправка запроса с помощью `fetch()`**:
   Функция `fetch()` отправляет POST-запрос на сервер по адресу `http://localhost:8080/log-grpc`, который мы определили в предыдущей лекции для обработки gRPC-запросов. Тело запроса содержит данные в формате JSON, а заголовки указывают на тип данных.

5. **Обработка ответа**:
Когда сервер вернет ответ, мы обновляем содержимое HTML-элементов `sent` и `received`, чтобы отобразить данные, отправленные и полученные в процессе запроса.
   
Когда сервер ответит, мы обрабатываем результат:
   - Если в ответе содержится ошибка, мы выводим её на экран.
   - Если всё прошло успешно, отображаем полученные данные и выводим сообщение о том, что запрос был выполнен успешно.

#### Пример из кода:
В результате добавления этого кода наш файл содержит новый блок для обработки gRPC-запроса:

```javascript
logGBtn.addEventListener("click", function() {
    const payload = {
        action: "log",
        log: {
            name: "event",
            data: "Some kind of gRPC data",
        }
    }

    const headers = new Headers();
    headers.append("Content-Type", "application/json");

    const body = {
        method: "POST",
        body: JSON.stringify(payload),
        headers: headers,
    }

    fetch("http://localhost:8080/log-grpc", body)
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
    })
})
```

#### Шаг 3: Исправление возможных ошибок
Во время работы с кодом важно обращать внимание на синтаксические ошибки и опечатки. В процессе написания кода в лекции преподаватель заметил мелкие ошибки и предложил их исправить. Например, может быть ошибка в названии переменной или в пути для запроса.

#### Заключение
После того, как мы добавили кнопку на фронтенде и прописали соответствующий обработчик событий, можно собрать проект и запустить сервер для тестирования gRPC. Мы проверим, работает ли всё корректно, и убедимся, что данные успешно отправляются и принимаются через gRPC.

В следующей лекции мы приступим к тестированию и отладке этих изменений.
