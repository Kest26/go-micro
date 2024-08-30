## Проверка работы микросервисов: практическое испытание

### Введение

Пришло время протестировать систему и выяснить, есть ли какие-то ошибки. Однако, перед тем как приступить к запуску, давайте проверим файлы проекта. В папке `handlers` я заметил опечатку в файле `handlers.go`, который находится в сервисе аутентификации. Убедитесь, что вы находитесь в корневой папке проекта, где должны быть файлы `docker-compose.yml` и `Makefile`.

### Компиляция и запуск контейнеров

Для начала давайте выполним команду:

```sh
make build_up
```

Эта команда соберет наши сервисы и поднимет Docker-контейнеры. Вероятно, в процессе могут возникнуть ошибки, связанные с опечатками, но посмотрим на результат.

**Результат в терминале:**

```sh
Building broker binary...
cd ../broker-service && env GOOS=linux CGO_ENABLED=0 go build -o brokerApp ./cmd/api
Done!
Building auth binary...
cd ../authentication-service && env GOOS=linux CGO_ENABLED=0 go build -o authApp ./cmd/api
Done!
Building logger binary...
cd ../logger-service && env GOOS=linux CGO_ENABLED=0 go build -o loggerServiceApp ./cmd/api
Done!
Stopping docker images (if running...)
docker-compose down
[+] Running 6/6
 ⠿ Container project-authentication-service-1  Removed                                                        3.3s
 ⠿ Container project-logger-service-1          Removed                                                        2.5s
 ⠿ Container project-broker-service-1          Removed                                                        2.7s
 ⠿ Container project-postgres-1                Removed                                                        3.8s
 ⠿ Container project-mongo-1                   Removed                                                        0.2s
 ⠿ Network project_default                     Removed                                                        0.5s
Building (when required) and starting docker images...
docker-compose up --build -d
[+] Building 7.1s (18/18) FINISHED

...

 ⠿ Network project_default                     Created                                                        0.4s
 ⠿ Container project-mongo-1                   Started                                                       17.4s
 ⠿ Container project-logger-service-1          Started                                                       14.9s
 ⠿ Container project-broker-service-1          Started                                                       15.3s
 ⠿ Container project-authentication-service-1  Started                                                       15.2s
 ⠿ Container project-postgres-1                Started                                                       16.2s
Docker images built and started!
```

Итак, все сервисы успешно собраны и запущены: Postgres, MongoDB и три наших микросервиса. Это хороший знак.

### Запуск фронтенда

Теперь давайте запустим фронтенд с помощью команды:

```sh
make start
```

**Результат:**

```sh
Building front end binary...
cd ../front-end && env CGO_ENABLED=0 go build -o frontApp ./cmd/web
Done!
Starting front end
cd ../front-end && ./frontApp &
Starting front end service on port 8079
```

Судя по выводу, ошибок в коде нет, и фронтенд успешно запустился. Давайте проверим это в браузере.

### Тестирование работы микросервисов

Откроем браузер и перейдем на `localhost`. Все сервисы запущены, давайте сначала проверим работу брокера, чтобы убедиться, что он работает. Брокер отвечает корректно — это уже хороший знак.

Теперь протестируем аутентификацию. Мы получили ответ, что говорит о том, что аутентификация прошла успешно и сервис, скорее всего, вызвал логирующий сервис. Давайте теперь протестируем логирование.

**Отправляем запрос:**

```json
{
    "action": "log",
    "log": {
        "name": "event",
        "data": "Some kind of data"
    }
}
```

**Получаем ответ:**

```json
{
    "error": false,
    "message": "logged"
}
```

В ответе указано, что логирование прошло успешно. Теперь мы можем проверить наличие записей в нашей MongoDB.

### Проверка записей в MongoDB

Для подключения к базе данных можно использовать MongoDB Compass. Это удобный клиент, который можно скачать по [ссылке](https://www.mongodb.com/try/download/compass).

https://www.mongodb.com/try/download/compass

После установки и запуска приложения, подключаемся к нашей базе данных.

Используем следующую строку подключения:

```
mongodb://admin:password@localhost:27017/logs?authSource=admin&readPreference=primary&appname=MongoDB%20Compass&directConnection=true&ssl=false
```

После подключения открываем базу данных `logs`. Мы должны увидеть две записи:

```json
{
  "_id": {
    "$oid": "66c5cfacf5c9a1184fc05601"
  },
  "name": "authentication",
  "data": "admin@example.com logged in",
  "created_at": {
    "$date": "2024-08-21T11:29:48.763Z"
  },
  "updated_at": {
    "$date": "2024-08-21T11:29:48.763Z"
  }
}

{
  "_id": {
    "$oid": "66c5cfc2f5c9a1184fc05602"
  },
  "name": "event",
  "data": "Some kind of data",
  "created_at": {
    "$date": "2024-08-21T11:30:10.097Z"
  },
  "updated_at": {
    "$date": "2024-08-21T11:30:10.097Z"
  }
}
```

Записи соответствуют нашим запросам, что подтверждает корректную работу системы.

### Заключение

Все сервисы заработали с первого раза, что весьма неожиданно. Теперь мы можем перейти к следующему микросервису.

---

### Разберем каждый параметр в строке подключения к MongoDB:

```
mongodb://admin:password@localhost:27017/logs?authSource=admin&readPreference=primary&appname=MongoDB%20Compass&directConnection=true&ssl=false
```

### 1. Протокол подключения

-   **`mongodb://`** - Протокол подключения, указывающий, что мы подключаемся к MongoDB.

### 2. Учетные данные пользователя

-   **`admin:password@`** - Учетные данные для авторизации:
    -   **`admin`** - Имя пользователя для подключения.
    -   **`password`** - Пароль для указанного пользователя.

### 3. Адрес и порт сервера

-   **`localhost:27017`** - Адрес сервера MongoDB:
    -   **`localhost`** - Хост, где запущена база данных (в данном случае, локальная машина).
    -   **`27017`** - Порт, на котором запущен сервер MongoDB (по умолчанию для MongoDB).

### 4. Имя базы данных

-   **`logs`** - Имя базы данных, к которой вы подключаетесь. Если она не существует, MongoDB создаст её при первом использовании.

### 5. Параметры подключения (указываются после `?`)

Эти параметры передаются в формате `ключ=значение`, разделенные `&`:

-   **`authSource=admin`** - Указывает базу данных, в которой находятся учетные данные пользователя. В данном случае это `admin`. Это важно, если вы хотите аутентифицироваться, но работать с другой базой данных.

-   **`readPreference=primary`** - Указывает предпочтительный реплицированный узел для чтения данных:

    -   **`primary`** - Чтение данных будет происходить с основного узла. Это стандартный режим для большинства случаев.

-   **`appname=MongoDB%20Compass`** - Имя приложения, которое подключается к MongoDB. Здесь указано `MongoDB Compass`, где `%20` заменяет пробел в URL-кодировке.

-   **`directConnection=true`** - Указывает, что соединение должно быть установлено напрямую с указанным узлом, а не через кластер (актуально, если вы работаете с одиночным инстансом MongoDB).

-   **`ssl=false`** - Указывает, что SSL/TLS шифрование при подключении не используется.
