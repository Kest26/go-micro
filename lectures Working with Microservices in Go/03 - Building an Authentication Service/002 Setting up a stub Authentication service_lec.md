### Создание и Настройка Микросервиса Аутентификации

#### Введение

В этой лекции мы начнем работу над микросервисом аутентификации. Процесс создания будет занимать некоторое время, но не является сложным. Мы создадим структуру проекта, настроим конфигурации и подготовим сервис к дальнейшему развитию.

#### Создание проекта

Первым шагом создадим папку для нашего сервиса аутентификации рядом с уже существующими папками `broker service` и `front end`. Это облегчит работу с `Makefile` и `Dockerfile`, если использовать одинаковую схему наименования. Назовем папку `authentication-service`.

Теперь добавим эту папку в рабочее пространство Visual Studio Code. Переходим в редактор и выбираем опцию "Add Folder to Workspace", затем указываем на созданную папку `authentication-service` и добавляем её в проект.

#### Инициализация Go-модуля

Откроем терминал и перейдем в директорию `authentication-service`. Выполним команду `go mod init`, чтобы инициализировать новый Go-модуль. Назовем его `authentication`. После этого у нас появится файл `go.mod`, который будет содержать информацию о модуле и его зависимостях.

#### Структура проекта

Создадим следующую структуру каталогов внутри папки `authentication-service`:

-   Папка `cmd` — содержит исполняемый файл.
-   Папка `api` внутри `cmd` — содержит точку входа для нашего API.

Создадим файл `main.go` в папке `api` с базовой структурой Go-программы:

```go
package main

import (
    "log"
)

const webPort = "80"

func main() {
    log.Println("Starting authentication service")
}
```

Этот файл будет служить точкой входа для нашего микросервиса.

#### Конфигурация приложения

Теперь создадим структуру для хранения конфигураций нашего приложения. Добавим новую структуру `Config` в `main.go`, которая будет содержать настройки подключения к базе данных и модели данных:

```go
type Config struct {
    DB     *sql.DB
    Models data.Models
}
```

#### Настройка данных

Создадим новую папку `data` в корне нашего сервиса и добавим файл `models.go`, который будет содержать структуры данных для работы с базой данных.

Для простоты в этом курсе мы предполагаем, что в базе данных будет лишь одна таблица `user`, но структура легко может быть расширена для поддержки дополнительных таблиц.

Код файла `models.go`:

```go
package data

import (
    "context"
    "database/sql"
    "errors"
    "log"
    "time"
    "golang.org/x/crypto/bcrypt"
)

const dbTimeout = time.Second * 3

var db *sql.DB

// New создаёт экземпляр пакета data и возвращает тип Models
func New(dbPool *sql.DB) Models {
    db = dbPool
    return Models{
        User: User{},
    }
}

// Models представляет доступные модели данных
type Models struct {
    User User
}

// User представляет структуру пользователя в базе данных
type User struct {
    ID        int       `json:"id"`
    Email     string    `json:"email"`
    FirstName string    `json:"first_name,omitempty"`
    LastName  string    `json:"last_name,omitempty"`
    Password  string    `json:"-"`
    Active    int       `json:"active"`
    CreatedAt time.Time `json:"created_at"`
    UpdatedAt time.Time `json:"updated_at"`
}
```

В этом файле определены базовые функции для работы с пользователями в базе данных, такие как добавление, обновление, удаление и выборка пользователей. Также реализованы функции для работы с паролями, включая их хэширование и проверку.

#### Настройка маршрутов (routes.go)

Теперь создадим файл `routes.go` в папке `cmd/api`, который будет содержать маршруты нашего API.

```go
package main

import (
	"net/http"

	"github.com/go-chi/chi/v5"
	"github.com/go-chi/cors"
)

func (*Config) routes() http.Handler {
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

	return mux
}
```

В этом файле создадим функцию `routes`, которая будет определять и возвращать маршрутизатор для обработки HTTP-запросов. Мы будем использовать библиотеку `chi` для маршрутизации и добавим middleware для обработки CORS (Cross-Origin Resource Sharing). Это необходимо, чтобы обеспечить доступ к нашему API из разных источников, включая веб-клиенты. Функция `routes` будет создавать новый маршрутизатор `chi.NewRouter()` и настраивать его с использованием `cors.Handler` с нужными опциями.

#### Запуск веб-сервера

Теперь можно настроить и запустить веб-сервер в `main.go`:

```go
func main() {
    log.Println("Starting authentication service")

    app := Config{}

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

Сервер будет слушать запросы на порту 80 и обрабатывать их с помощью заранее настроенных маршрутов.

#### Заключение

Теперь у нас есть базовая структура микросервиса аутентификации. В следующей лекции мы добавим функциональность для подключения к базе данных и продолжим разработку нашего сервиса.
