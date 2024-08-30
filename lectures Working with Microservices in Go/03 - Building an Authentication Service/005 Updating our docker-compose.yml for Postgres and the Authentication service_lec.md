### Настройка и запуск базы данных Postgres и микросервиса аутентификации

В этом уроке мы рассмотрим процесс настройки базы данных Postgres и микросервиса аутентификации. Мы исправим ошибки в коде, настроим `docker-compose.yml`, создадим Dockerfile для аутентификации и обновим Makefile для сборки и запуска всех сервисов.

#### 1. Исправление ошибок в коде

Прежде чем приступить к настройке базы данных и Docker, нужно исправить несколько ошибок в файле `main.go` внутри сервиса аутентификации.

-   **Ошибка с импортом**: В импортируемой библиотеке пропущена буква "C". Это нужно исправить в следующей строке:

    ```go
    import "github.com/somepackage"
    ```

-   **Лишняя скобка**: В функции `OpenDB()` на 57-й строке присутствует лишняя закрывающая скобка, которую нужно удалить:

    ```go
    db, err := sql.Open("postgres", dsn))
    ```

    Исправленный код должен выглядеть так:

    ```go
    db, err := sql.Open("postgres", dsn)
    ```

#### 2. Настройка базы данных Postgres в Docker Compose

Теперь мы добавим конфигурацию для базы данных Postgres в файл `docker-compose.yml`, который находится в корневой папке проекта.

-   **Создание сервиса Postgres**: Добавляем новый сервис `postgres`, который использует образ `postgres:14.0`:

    ```yaml
    postgres:
        image: "postgres:14.0"
        ports:
            - "5432:5432"
        restart: always
        deploy:
            mode: replicated
            replicas: 1
        environment:
            POSTGRES_USER: postgres
            POSTGRES_PASSWORD: password
            POSTGRES_DB: users
        volumes:
            - ./db-data/postgres/:/var/lib/postgresql/data/
    ```

-   **Порты**: Локальный порт `5432` будет связан с портом `5432` внутри контейнера Docker. Это позволяет подключаться к базе данных как из Docker, так и с локальной машины.

-   **Переменные окружения**: Настраиваем следующие переменные окружения:

    -   `POSTGRES_USER`: Имя пользователя базы данных (`postgres`).
    -   `POSTGRES_PASSWORD`: Пароль для пользователя (`password`).
    -   `POSTGRES_DB`: Имя базы данных (`users`), которая будет создана при первом запуске.

-   **Том для данных**: Настраиваем том для хранения данных базы:

    ```yaml
    volumes:
        - ./db-data/postgres/:/var/lib/postgresql/data/
    ```

    Путь `./db-data/postgres/` на локальной машине будет монтироваться в контейнер в директорию `/var/lib/postgresql/data/`. Это обеспечивает сохранение данных между перезапусками контейнера.

-   **Создание необходимых папок**: Создайте папку `db-data` в корневой папке проекта и внутри неё папку `postgres`. Docker будет использовать её для хранения данных базы.

#### 3. Настройка сервиса аутентификации в Docker Compose

Далее добавим конфигурацию для сервиса аутентификации в `docker-compose.yml`.

-   **Создание сервиса аутентификации**: Добавляем сервис `authentication-service`, который будет создавать Docker-образ из указанного контекста:

    ```yaml
    authentication-service:
        build:
            context: ./../authentication-service
            dockerfile: ./../authentication-service/authentication-service.dockerfile
        restart: always
        ports:
            - "8081:80"
        deploy:
            mode: replicated
            replicas: 1
        environment:
            DSN: "host=postgres port=5432 user=postgres password=password dbname=users sslmode=disable timezone=UTC connect_timeout=5"
    ```

-   **Порты**: Настраиваем порт `8081` на локальной машине, который будет связан с портом `80` внутри Docker-контейнера.

-   **Переменные окружения**: Добавляем переменную `DSN`, которая содержит строку подключения к базе данных. Эта строка включает в себя:
    -   Хост: `host=postgres` (название сервиса базы данных).
    -   Порт: `port=5432`.
    -   Пользователь: `user=postgres`.
    -   Пароль: `password=password`.
    -   Имя базы данных: `dbname=users`.
    -   SSL-режим: `sslmode=disable` (отключение SSL для простоты).
    -   Временная зона: `timezone=UTC`.
    -   Тайм-аут подключения: `connect_timeout=5`.

#### 4. Создание Dockerfile для сервиса аутентификации

Теперь создадим Dockerfile для сервиса аутентификации.

-   **Создание файла Dockerfile**: В папке `authentication-service` создайте новый файл `authentication-service.dockerfile` и добавьте следующий код:

    ```dockerfile
    FROM alpine:latest

    RUN mkdir /app

    COPY authApp /app

    CMD [ "/app/authApp" ]
    ```

-   **Объяснение**:
    -   **FROM alpine:latest**: Используем легковесный образ Alpine Linux.
    -   **RUN mkdir /app**: Создаём директорию `/app` внутри контейнера.
    -   **COPY authApp /app**: Копируем скомпилированное приложение `authApp` в директорию `/app`.
    -   **CMD [ "/app/authApp" ]**: Указываем команду для запуска приложения внутри контейнера.

#### 5. Обновление Makefile для сборки и запуска сервисов

Теперь обновим Makefile, чтобы он поддерживал сборку и запуск всех сервисов.

-   **Добавление команд для сборки аутентификации**:

    ```makefile
    ## build_auth: builds the auth binary as a linux executable
    build_auth:
    	@echo "Building auth binary..."
    	cd ../authentication-service && env GOOS=linux CGO_ENABLED=0 go build -o ${AUTH_BINARY} ./cmd/api
    	@echo "Done!"
    ```

-   **Команда для сборки и запуска всех сервисов**:

    ```makefile
    ## up_build: stops docker-compose (if running), builds all projects and starts docker compose
    up_build: build_broker build_auth
    	@echo "Stopping docker images (if running...)"
    	docker-compose down
    	@echo "Building (when required) and starting docker images..."
    	docker-compose up --build -d
    	@echo "Docker images built and started!"
    ```

-   **Переменные для хранения имён бинарных файлов**:

    ```makefile
    AUTH_BINARY=authApp
    ```

#### 6. Запуск и проверка сервисов

Теперь, когда все файлы настроены, можно попробовать запустить сервисы.

1. **Сборка и запуск сервисов**: В корневой папке проекта выполните команду:

    ```bash
    make up_build
    ```

    Эта команда соберёт и запустит все сервисы.

2. **Проверка ошибок**: Если возникнут ошибки, например, отсутствующие импорты (например, `fmt` или `github.com/go-chi/cors`), добавьте их в код и снова запустите команду.

3. **Проверка состояния сервисов через Docker Desktop**: После успешного запуска откройте Docker Desktop и убедитесь, что все сервисы работают корректно. Возможна небольшая задержка при первом запуске Postgres, так как потребуется время на инициализацию базы данных.

#### 7. Следующие шаги

Теперь, когда сервис аутентификации запущен и подключён к базе данных Postgres, следующим шагом будет создание таблиц в базе и добавление данных для тестирования функциональности аутентификации. Мы займёмся этим в следующей лекции.
