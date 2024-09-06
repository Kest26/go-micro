## Обновление брокера для взаимодействия с RabbitMQ

**Введение**

В этом шаге мы обновим наш сервис брокера для обработки запросов, связанных с отправкой данных в RabbitMQ. Брокер **не будет напрямую взаимодействовать с `listener-service`**; вместо этого он будет отправлять информацию в RabbitMQ, откуда `listener-service` будет ее извлекать для дальнейшей обработки.

**1. Импорт драйвера RabbitMQ**

Первым шагом будет импорт драйвера RabbitMQ в наш сервис брокера. Мы остановим текущий запущенный процесс в терминале, перейдем в директорию `broker-service`, и установим необходимый пакет с помощью следующей команды:

```bash
go get github.com/rabbitmq/amqp091-go
```

После установки пакета, мы добавим его в раздел импортов в файле `main.go` в директории `broker-service/cmd/api`. Мы создадим алиас `amqp`, чтобы упростить доступ к функционалу RabbitMQ:

```go
import (
	"fmt"
	"log"
	"math"
	"net/http"
	"os"
	"time"

	amqp "github.com/rabbitmq/amqp091-go"
)
```

Этот шаг подготовит наш проект к работе с RabbitMQ.

**2. Добавление подключения к RabbitMQ в конфигурацию**

Теперь, когда драйвер RabbitMQ импортирован, нужно настроить подключение к RabbitMQ. Для этого мы обновим структуру `Config`, добавив в нее новое поле `Rabbit`, которое будет указателем на объект `amqp.Connection`. Это соединение позволит нам взаимодействовать с RabbitMQ в других частях программы.

Вот как будет выглядеть обновленная структура `Config` в файле `main.go`:

```go
type Config struct {
	Rabbit *amqp.Connection
}
```

Следующим шагом будет создание функции для подключения к RabbitMQ. Мы добавим функцию `connect`, которая будет пытаться установить соединение с RabbitMQ с использованием экспоненциальной задержки в случае неудачи:

```go
func connect() (*amqp.Connection, error) {
	var counts int64
	var backOff = 1 * time.Second
	var connection *amqp.Connection

	// не продолжаем, пока RabbitMQ не готов
	for {
		c, err := amqp.Dial("amqp://guest:guest@rabbitmq")
		if err != nil {
			fmt.Println("RabbitMQ не готов...")
			counts++
		} else {
			log.Println("Подключено к RabbitMQ!")
			connection = c
			break
		}

		if counts > 5 {
			fmt.Println(err)
			return nil, err
		}

		backOff = time.Duration(math.Pow(float64(counts), 2)) * time.Second
		log.Println("Откат...")
		time.Sleep(backOff)
		continue
	}

	return connection, nil
}
```

В этой функции мы используем цикл `for`, чтобы повторять попытки подключения к RabbitMQ до тех пор, пока оно не будет успешно установлено, или не истечет количество попыток.

Далее мы используем эту функцию в `main`, чтобы установить соединение при запуске брокера:

```go
func main() {
	// пытаемся подключиться к RabbitMQ
	rabbitConn, err := connect()
	if err != nil {
		log.Println(err)
		os.Exit(1)
	}
	defer rabbitConn.Close()

	app := Config{
		Rabbit: rabbitConn,
	}

	log.Printf("Запуск брокера на порту %s\n", webPort)

	// определяем HTTP сервер
	srv := &http.Server{
		Addr:    fmt.Sprintf(":%s", webPort),
		Handler: app.routes(),
	}

	// запускаем сервер
	err = srv.ListenAndServe()
	if err != nil {
		log.Panic(err)
	}
}
```

**3. Перенос файлов для работы с событиями**

Для работы с событиями, передаваемыми через RabbitMQ, мы можем воспользоваться кодом из Listener service. Создадим в `broker-service` новую папку `event`, а затем скопируем файлы `event.go` и `consumer.go` из Listener service в эту папку.

```go
// event/event.go
package event

import (
	amqp "github.com/rabbitmq/amqp091-go"
)

func declareExchange(ch *amqp.Channel) error {
	return ch.ExchangeDeclare(
		"logs_topic", // имя
		"topic",      // тип
		true,         // долговечность
		false,        // автозакрытие
		false,        // внутренний
		false,        // без ожидания
		nil,          // аргументы
	)
}

func declareRandomQueue(ch *amqp.Channel) (amqp.Queue, error) {
	return ch.QueueDeclare(
		"",        // имя
		false,     // долговечность
		false,     // удаление, когда не используется
		true,      // эксклюзивность
		false,     // без ожидания
		nil,       // аргументы
	)
}
```

В этом файле мы объявляем обменник и случайную очередь для обработки сообщений. Эта структура будет полезна, когда брокер начнет отправлять события в RabbitMQ.

```go
// event/consumer.go
package event

import (
	"bytes"
	"encoding/json"
	"fmt"
	"log"
	"net/http"

	amqp "github.com/rabbitmq/amqp091-go"
)

type Consumer struct {
	conn      *amqp.Connection
	queueName string
}

func NewConsumer(conn *amqp.Connection) (Consumer, error) {
	consumer := Consumer{
		conn: conn,
	}

	err := consumer.setup()
	if err != nil {
		return Consumer{}, err
	}

	return consumer, nil
}

func (consumer *Consumer) setup() error {
	channel, err := consumer.conn.Channel()
	if err != nil {
		return err
	}

	return declareExchange(channel)
}

type Payload struct {
	Name string `json:"name"`
	Data string `json:"data"`
}

func (consumer *Consumer) Listen(topics []string) error {
	ch, err := consumer.conn.Channel()
	if err != nil {
		return err
	}
	defer ch.Close()

	q, err := declareRandomQueue(ch)
	if err != nil {
		return err
	}

	for _, s := range topics {
		ch.QueueBind(
			q.Name,
			s,
			"logs_topic",
			false,
			nil,
		)

		if err != nil {
			return err
		}
	}

	messages, err := ch.Consume(q.Name, "", true, false, false, false, nil)
	if err != nil {
		return err
	}

	forever := make(chan bool)
	go func() {
		for d := range messages {
			var payload Payload
			_ = json.Unmarshal(d.Body, &payload)

			go handlePayload(payload)
		}
	}()

	fmt.Printf("Ожидание сообщений [Exchange, Queue] [logs_topic, %s]\n", q.Name)
	<-forever

	return nil
}

func handlePayload(payload Payload) {
	switch payload.Name {
	case "log", "event":
		err := logEvent(payload)
		if err != nil {
			log.Println(err)
		}

	case "auth":
		// аутентификация

	default:
		err := logEvent(payload)
		if err != nil {
			log.Println(err)
		}
	}
}

func logEvent(entry Payload) error {
	jsonData, _ := json.MarshalIndent(entry, "", "\t")

	logServiceURL := "http://logger-service/log"

	request, err := http.NewRequest("POST", logServiceURL, bytes.NewBuffer(jsonData))
	if err != nil {
		return err
	}

	request.Header.Set("Content-Type", "application/json")

	client := &http.Client{}

	response, err := client.Do(request)
	if err != nil {
		return err
	}
	defer response.Body.Close()

	if response.StatusCode != http.StatusAccepted {
		return err
	}

	return nil
}
```

Файл `consumer.go` управляет созданием и настройкой `Consumer`, который будет прослушивать сообщения из очереди RabbitMQ и обрабатывать их в зависимости от типа данных.

**4. Обновление обработки запросов на логирование**

Теперь, когда у нас есть все необходимые компоненты, мы обновим обработку запросов на логирование в брокере, чтобы данные направлялись в RabbitMQ вместо вызова логирования напрямую.

В файле `routes.go`, основной маршрут `/handle` направляется на функцию `HandleSubmission`, которая находится в файле `handlers.go`:

```go
func (app *Config) HandleSubmission(w http.ResponseWriter, r *http.Request) {
	var requestPayload RequestPayload

	err := app.readJSON(w, r, &requestPayload)
	if err != nil {
		app.errorJSON(w, err)
		return
	}

	switch requestPayload.Action {
	case "auth":
		app.authenticate(w, requestPayload.Auth)
	case "log":
		app.logItem(w, requestPayload.Log)
	case "mail":
		app.sendMail(w, requestPayload.Mail)
	default:
		app.errorJSON(w, errors.New("unknown action"))
	}
}
```

Когда мы получаем запрос `"log"`, мы вызываем `app.logItem`, то есть вызваем функцию `logItem` и напрямую отправляем данные в `logger-service`. Пока мы оставим `logItem` в коде в качестве примера (справки), но вместо неё потом добавим новую функцию, которая будет направлять данные в очередь RabbitMQ.

Теперь, когда будет поступать запрос на логирование, `HandleSubmission` направит данные в новую функцию, которая отправит данные в очередь RabbitMQ, а после этого данные будут переданы в микросервис `listener-service`, который сможет их забрать и обработать.

Но к этому мы приступим вследующей лекции.
