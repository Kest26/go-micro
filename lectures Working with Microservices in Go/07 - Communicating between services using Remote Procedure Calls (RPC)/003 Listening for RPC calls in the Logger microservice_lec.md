## Настройка прослушивания RPC-запросов в микросервисе логирования

---

### 1. **Введение: Запуск RPC-сервера**

Мы уже реализовали сервер RPC и создали тип для получения полезной нагрузки, а также метод `LogInfo`, доступный для вызова через RPC. Теперь нам нужно фактически запустить этот сервер. Для этого мы вернемся к файлу `main.go` в папке `cmd/api` нашего микросервиса логирования.

В начале файла мы уже определили константу для порта RPC, который равен 5001. Следующим шагом будет написание функции для прослушивания входящих RPC-соединений.

Пример константы порта RPC:

```go
const (
    rpcPort = "5001"
)
```

### 2. **Создание функции для прослушивания RPC-запросов**

Для реализации прослушивания RPC-запросов мы создаем новую функцию `rpcListen`, которая будет привязана к структуре `Config`. Эта функция будет отвечать за запуск RPC-сервера на порту, определенном ранее.

Внутри функции первым шагом будет вывод в консоль сообщения о запуске RPC-сервера:

```go
func (app *Config) rpcListen() error {
    log.Println("Starting RPC server on port ", rpcPort)
```

Далее мы создаем переменную `listen` для прослушивания TCP-соединений с использованием стандартной библиотеки `net`:

```go
listen, err := net.Listen("tcp", fmt.Sprintf("0.0.0.0:%s", rpcPort))
if err != nil {
    return err
}
defer listen.Close()
```

Мы используем метод `net.Listen` для прослушивания всех IP-адресов (`0.0.0.0`) на порту `5001`, указанном в переменной `rpcPort`. В случае ошибки сервер вернет ошибку и прекратит работу.

### 3. **Цикл обработки входящих соединений**

После установки прослушивания создается бесконечный цикл, который будет принимать входящие соединения и передавать их на обработку. Каждое соединение будет обрабатываться в отдельной горутине, чтобы сервер мог продолжать принимать новые запросы.

Вот код, который реализует прием и обработку входящих соединений:

```go
for {
    rpcConn, err := listen.Accept()
    if err != nil {
        continue
    }
    go rpc.ServeConn(rpcConn)
}
```

- В переменной `rpcConn` сохраняется новое соединение.
- Если при приеме соединения возникает ошибка, выполнение возвращается в начало цикла.
- Для каждого нового соединения создается отдельная горутина, в которой вызов `rpc.ServeConn(rpcConn)` обрабатывает это соединение.

### 4. **Регистрация RPC-сервера**

Чтобы сервер мог принимать и обрабатывать RPC-запросы, нам необходимо зарегистрировать тип `RPCServer` в стандартной библиотеке RPC Go. Регистрация выполняется с помощью метода `rpc.Register`, который привязывает наши методы к RPC-серверу.

Регистрация происходит в функции `main` следующим образом:

```go
err = rpc.Register(new(RPCServer))
if err != nil {
    log.Panic("Failed to register RPC server:", err)
}
```

Эта строка регистрирует новый экземпляр структуры `RPCServer`, делая методы этого типа доступными для вызова через RPC.

### 5. **Запуск RPC-сервера в главной функции**

После того как мы создали и зарегистрировали RPC-сервер, нам необходимо его запустить. Для этого мы вызываем нашу функцию `rpcListen` в горутине прямо перед запуском веб-сервера:

```go
go app.rpcListen()
```

Таким образом, сервер RPC будет запущен параллельно с веб-сервером и начнет принимать RPC-запросы на порту 5001.

### 6. **Заключение: Подготовка к отправке RPC-запросов**

На этом этапе мы настроили микросервис логирования для приема RPC-запросов. Теперь сервер может обрабатывать входящие соединения, принимать полезную нагрузку и выполнять метод `LogInfo` для записи данных в MongoDB. Следующим шагом будет настройка брокера для отправки RPC-запросов, что мы рассмотрим в следующих лекциях.

---

## Полный код файла `main.go`:

```go
package main

import (
	"context"
	"fmt"
	"log"
	"log-service/data"
	"net"
	"net/http"
	"net/rpc"
	"time"

	"go.mongodb.org/mongo-driver/mongo"
	"go.mongodb.org/mongo-driver/mongo/options"
)

const (
	webPort  = "80"
	rpcPort  = "5001"
	mongoURL = "mongodb://mongo:27017"
	gRpcPort = "50001"
)

var client *mongo.Client

type Config struct {
	Models data.Models
}

func main() {
	// Подключение к MongoDB
	mongoClient, err := connectToMongo()
	if err != nil {
		log.Panic(err)
	}
	client = mongoClient

	// Создание контекста для отключения от базы данных
	ctx, cancel := context.WithTimeout(context.Background(), 15*time.Second)
	defer cancel()

	// Закрытие соединения с MongoDB
	defer func() {
		if err = client.Disconnect(ctx); err != nil {
			panic(err)
		}
	}()

	app := Config{
		Models: data.New(client),
	}

	// Регистрация RPC-сервера
	err = rpc.Register(new(RPCServer))
	if err != nil {
		log.Panic("Failed to register RPC server:", err)
	}

	// Запуск RPC-сервера в отдельной горутине
	go app.rpcListen()

	// Запуск веб-сервера
	log.Println("Starting service on port", webPort)
	srv := &http.Server{
		Addr:    fmt.Sprintf(":%s", webPort),
		Handler: app.routes(),
	}

	err = srv.ListenAndServe()
	if err != nil {
		log.Panic(err)
	}
}

func (app *Config) rpcListen() error {
	log.Println("Starting RPC server on port ", rpcPort)
	listen, err := net.Listen("tcp", fmt.Sprintf("0.0.0.0:%s", rpcPort))
	if err != nil {
		return err
	}
	defer listen.Close()

	for {
		rpcConn, err := listen.Accept()
		if err != nil {
			continue
		}
		go rpc.ServeConn(rpcConn)
	}
}

func connectToMongo() (*mongo.Client, error) {
	// Создание опций для подключения
	clientOptions := options.Client().ApplyURI(mongoURL)
	clientOptions.SetAuth(options.Credential{
		Username: "admin",
		Password: "password",
	})

	// Подключение к MongoDB
	c, err := mongo.Connect(context.TODO(), clientOptions)
	if err != nil {
		log.Println("Error connecting:", err)
		return nil, err
	}

	log.Println("Connected to mongo!")

	return c, nil
}
```
