## Создание Docker-образа и обновление Makefile

### Введение

В этой лекции мы будем создавать Dockerfile для нового сервиса `listener` и обновлять Makefile, чтобы добавить поддержку сборки и запуска этого сервиса. Вместе с тем мы рассмотрим процесс добавления нового сервиса в `docker-compose.yml`, что позволит интегрировать его в текущую архитектуру микросервисов.

### Создание Dockerfile для сервиса `listener`

Первым шагом будет создание Dockerfile для нашего сервиса `listener`. В качестве шаблона мы воспользуемся уже существующим Dockerfile из другого сервиса. Это позволит нам ускорить процесс и избежать типичных ошибок.

1. **Поиск и копирование Dockerfile:**
   - Для начала откроем Dockerfile, используемый другим сервисом. Например, давайте возьмем Dockerfile из сервиса `broker`.
   - Откроем его, выделим весь текст и скопируем его содержимое.

   Пример кода из Dockerfile сервиса `broker`:
   ```dockerfile
   FROM alpine:latest

   RUN mkdir /app

   COPY brokerApp /app

   CMD [ "/app/brokerApp" ]
   ```

2. **Создание нового Dockerfile для сервиса `listener`:**
   - В директории сервиса `listener` создадим новый файл и назовем его `listener-service.dockerfile`.
   - Вставим туда скопированный код из Dockerfile сервиса `broker`.

3. **Редактирование Dockerfile:**
   - После вставки кода нам нужно внести несколько изменений, чтобы этот файл был адаптирован для нашего нового сервиса.
   - Изменим название приложения в строке `COPY` с `brokerApp` на `listenerApp`, чтобы соответствовать бинарному файлу, который будет создан для сервиса `listener`.
   - Аналогично, изменим название в строке `CMD`, чтобы запускался наш новый бинарный файл `listenerApp`.

   Пример итогового Dockerfile для сервиса `listener`:
   ```dockerfile
   FROM alpine:latest

   RUN mkdir /app

   COPY listenerApp /app

   CMD [ "/app/listenerApp" ]
   ```

Таким образом, наш Dockerfile готов. Поскольку наше приложение представляет собой один бинарный файл, дополнительные настройки не требуются.

### Обновление Makefile для поддержки нового сервиса

Следующим шагом будет обновление Makefile, чтобы добавить задачу для сборки и запуска сервиса `listener`.

1. **Добавление задачи для сборки `listener`:**
   - Откроем файл `Makefile`.
   - Найдем задачу, связанную с другим сервисом, например, `build_logger`. Скопируем весь блок кода этой задачи.

     Пример задачи для сборки логгера:
     ```makefile
     ## build_logger: builds the logger binary as a linux executable
     build_logger:
     	@echo "Building logger binary..."
     	cd ../logger-service && env GOOS=linux CGO_ENABLED=0 go build -o ${LOGGER_BINARY} ./cmd/api
     	@echo "Done!"
     ```

   - Вставим скопированную задачу ниже и изменим название задачи на `build_listener`. 

     Пример обновленной задачи:
     ```makefile
     ## build_listener: builds the listener binary as a linux executable
     build_listener:
     	@echo "Building listener binary..."
     	cd ../listener-service && env GOOS=linux CGO_ENABLED=0 go build -o ${LISTENER_BINARY} .
     	@echo "Done!"
     ```

2. **Обновление общей задачи сборки:**
   - Найдем задачу `up_build`, которая отвечает за сборку и запуск всех сервисов, и добавим туда вызов новой задачи `build_listener`.

     Пример изменения задачи `up_build`:
     ```makefile
     ## up_build: stops docker-compose (if running), builds all projects and starts docker compose
     up_build: build_broker build_auth build_logger build_mail build_listener
     	@echo "Stopping docker images (if running...)"
     	docker-compose down
     	@echo "Building (when required) and starting docker images..."
     	docker-compose up --build -d
     	@echo "Docker images built and started!"
     ```

Теперь наш Makefile поддерживает сборку и запуск нового сервиса `listener`.

### Обновление Docker Compose для интеграции нового сервиса

Чтобы интегрировать новый сервис `listener` в нашу архитектуру, необходимо добавить его в файл `docker-compose.yml`.

1. **Добавление нового сервиса в `docker-compose.yml`:**
   - Откроем файл `docker-compose.yml`, который расположен в корневой директории проекта.
   - Найдем секцию, где описаны другие сервисы, например, `broker-service`.
   - Скопируем эту секцию и вставим ее ниже, изменив название на `listener-service` и указав путь к Dockerfile и контексту сборки.

     Пример добавления нового сервиса:
     ```yml
     listener-service:
       build:
         context: ./../listener-service
         dockerfile: ./../listener-service/listener-service.dockerfile
       deploy:
         mode: replicated
         replicas: 1
     ```

2. **Проверка корректности изменений:**
   - Убедитесь, что все изменения сохранены и что нет лишних пробелов или опечаток.

### Проверка работы и устранение ошибок

Теперь, когда все изменения внесены, можно проверить работу нового сервиса.

1. **Сборка и запуск сервисов:**
   - Откроем терминал, перейдем в корневую папку проекта и выполните команду:

     ```bash
     make up_build
     ```

   - Эта команда остановит все запущенные контейнеры (если они работают), соберет новые образы и запустит их заново.

2. **Проверка на ошибки:**
   - Если система выдает ошибку, например, не может найти бинарный файл `listenerApp`, проверьте Makefile и Dockerfile на наличие опечаток или неправильных путей. Убедитесь, что в Makefile и Dockerfile указаны правильные имена файлов и директории.

   Пример исправления ошибки в Makefile:
   ```makefile
   ## build_listener: builds the listener binary as a linux executable
   build_listener:
     	@echo "Building listener binary..."
     	cd ../listener-service && env GOOS=linux CGO_ENABLED=0 go build -o ${LISTENER_BINARY} .
     	@echo "Done!"
   ```

3. **Повторная сборка:**
   - После исправления ошибок, выполните команду `make up_build` снова и проверьте, работает ли сервис корректно.

### Заключение

Теперь, когда мы добавили новый сервис `listener`, обновили Makefile и Docker Compose, и успешно собрали проект, мы готовы к следующему шагу. На следующем этапе мы внесем изменения в микросервис `broker` и фронтенд, чтобы проверить работу всей системы в целом.
