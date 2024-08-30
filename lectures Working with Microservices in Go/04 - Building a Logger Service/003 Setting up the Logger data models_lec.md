### Работа с MongoDB в Go: Установка соединения и создание моделей данных

#### Введение

В этом уроке мы продолжим изучение работы с MongoDB на языке Go. Мы рассмотрим, как установить соединение с базой данных MongoDB, как корректно завершить это соединение при завершении работы программы, а также как определить и использовать модели данных для работы с MongoDB. Важной частью будет понимание концепций, таких как контексты в Go, а также взаимодействие с коллекциями MongoDB.

#### Настройка проекта и подключение к MongoDB

Прежде чем мы начнем работу с моделями данных, необходимо убедиться, что наше приложение Go успешно подключается к базе данных MongoDB.

1. **Создание и конфигурация проекта:**

   - Убедитесь, что у вас уже установлен пакет `go.mongodb.org/mongo-driver/mongo`. Если это не так, добавьте его в проект с помощью команды:
     ```bash
     go get go.mongodb.org/mongo-driver/mongo
     ```
   
   - Основной файл нашего проекта будет называться `main.go`, который будет содержать логику для подключения к MongoDB и управления жизненным циклом подключения.

2. **Инициализация и подключение к MongoDB:**

   В файле `main.go` сначала создаем необходимые константы и переменные:

   ```go
   package main

   import (
       "context"
       "log"
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
   }
   ```

   - **Константы:** Они включают URL для подключения к MongoDB, а также порты, используемые вашим приложением.
   - **Глобальная переменная `client`:** Это будет указатель на клиент MongoDB, через который будет происходить взаимодействие с базой данных.

3. **Функция `main`:**

   Функция `main` в Go — это точка входа в ваше приложение. Здесь мы сначала подключаемся к MongoDB, а затем обеспечиваем корректное завершение этого подключения, когда приложение завершает свою работу:

   ```go
   func main() {
       // Подключение к MongoDB
       mongoClient, err := connectToMongo()
       if err != nil {
           log.Panic(err)
       }

       client = mongoClient

       // Создание контекста для отключения от MongoDB
       ctx, cancel := context.WithTimeout(context.Background(), 15*time.Second)
       defer cancel()

       // Закрытие подключения при завершении программы
       defer func() {
           if err = client.Disconnect(ctx); err != nil {
               panic(err)
           }
       }()
   }
   ```

   - **Подключение к MongoDB:** Вызов функции `connectToMongo`, которая возвращает клиент для работы с MongoDB. В случае ошибки мы вызываем `log.Panic`, чтобы немедленно завершить работу программы с выводом ошибки.
   - **Создание контекста:** Контекст `context.WithTimeout` устанавливает ограничение по времени (15 секунд) на выполнение операций, связанных с отключением от базы данных. Функция `cancel` отменяет контекст, когда операция завершается.
   - **Корректное закрытие подключения:** Используя оператор `defer`, мы откладываем выполнение кода до завершения функции `main`, что позволяет гарантировать закрытие подключения даже при возникновении ошибок.

4. **Функция подключения к MongoDB `connectToMongo`:**

   В этой функции мы создаем параметры подключения, устанавливаем авторизационные данные и выполняем подключение:

   ```go
   func connectToMongo() (*mongo.Client, error) {
       // Создание опций подключения
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

       return c, nil
   }
   ```

   - **Опции подключения:** Мы используем метод `ApplyURI` для задания URL MongoDB и добавляем авторизационные данные с помощью метода `SetAuth`.
   - **Подключение:** Метод `mongo.Connect` выполняет подключение с заданными параметрами. В случае успешного подключения возвращаем объект клиента, иначе логируем ошибку и возвращаем `nil`.

#### Создание и использование моделей данных

Теперь, когда наше приложение подключено к MongoDB, нам нужно определить модели данных, которые будут использоваться для взаимодействия с коллекциями базы данных.

1. **Создание файла моделей данных:**

   Все наши модели данных будут находиться в файле `models.go`, который создается в папке `data`. 

   ```go
   package data

   import (
       "context"
       "log"
       "time"

       "go.mongodb.org/mongo-driver/bson"
       "go.mongodb.org/mongo-driver/mongo"
       "go.mongodb.org/mongo-driver/mongo/options"
   )
   ```

   - **Пакет `data`:** Этот пакет будет содержать все структуры и функции, связанные с данными, которые мы будем использовать в приложении.

2. **Объявление переменных и создание структур:**

   Внутри `models.go` сначала создадим глобальную переменную `client`, которая будет хранить указатель на клиент MongoDB:

   ```go
   var client *mongo.Client
   ```

   Затем создаем структуру `Models`, которая будет включать в себя все модели данных, с которыми мы будем работать:

   ```go
   type Models struct {
       LogEntry LogEntry
   }
   ```

   - **`LogEntry`:** Это модель данных, которая будет использоваться для хранения логов в MongoDB.

3. **Создание функции `New`:**

   Функция `New` инициализирует переменную `client` и возвращает экземпляр структуры `Models`:

   ```go
   func New(mongo *mongo.Client) Models {
       client = mongo

       return Models{
           LogEntry: LogEntry{},
       }
   }
   ```

   - **Инициализация клиента:** Мы сохраняем указатель на клиент MongoDB в глобальной переменной `client` и возвращаем структуру `Models`, которая будет использоваться для доступа к методам, связанным с логами.

4. **Определение структуры `LogEntry`:**

   Структура `LogEntry` описывает документ, который будет храниться в коллекции MongoDB:

   ```go
   type LogEntry struct {
       ID        string    `bson:"_id,omitempty" json:"id,omitempty"`
       Name      string    `bson:"name" json:"name"`
       Data      string    `bson:"data" json:"data"`
       CreatedAt time.Time `bson:"created_at" json:"created_at"`
       UpdatedAt time.Time `bson:"updated_at" json:"updated_at"`
   }
   ```

   - **Поля структуры:** Поля структуры соответствуют полям, которые будут храниться в коллекции MongoDB. 
     - `ID`: Уникальный идентификатор документа.
     - `Name`: Название лог-записи.
     - `Data`: Основное содержание лог-записи.
     - `CreatedAt` и `UpdatedAt`: Метки времени создания и обновления записи.
   - **Теги `bson` и `json`:** Эти теги указывают, как поля структуры должны быть сериализованы при записи в MongoDB и при работе с JSON-данными. Например, поле `ID` будет называться `_id` в MongoDB и `id` в JSON.

5. **Функция для вставки данных `Insert`:**

   Эта функция позволяет добавлять новый документ в коллекцию MongoDB:

   ```go
   func (l *LogEntry) Insert(entry LogEntry) error {
       collection := client.Database("logs").Collection("logs")

       _, err := collection.InsertOne(context.TODO(), LogEntry{
           Name:      entry.Name,
           Data:      entry.Data,
           CreatedAt: time.Now(),
           UpdatedAt: time.Now(),
       })
       if err != nil {
           log.Println("Error inserting into logs:", err)
           return err
       }

       return nil
   }
   ```

   - **Получение коллекции:** Мы обращаемся к коллекции `logs` в базе данных, где будут храниться наши лог-записи.
   - **Вставка документа:** Метод `InsertOne` добавляет новый документ в коллекцию. Мы передаем контекст `context.TODO()` и новую запись с заполненными полями. Если возникает ошибка, она логируется и возвращается из функции.

#### Получение данных из MongoDB

Теперь рассмотрим, как извлечь все записи из коллекции.

1. **Функция для получения всех записей `All`:**

   Эта функция возвращает все записи, хранящиеся в коллекции:

   ```go
   func (l *LogEntry) All() ([]*LogEntry, error) {
       ctx, cancel := context.WithTimeout(context.Background(), 15*time.Second)
       defer cancel()

       collection := client.Database("logs").Collection("logs")



       opts := options.Find().SetSort(bson.D{{Key: "created_at", Value: -1}})
       cursor, err := collection.Find(context.TODO(), bson.D{}, opts)
       if err != nil {
           log.Println("Error finding all logs:", err)
           return nil, err
       }
       defer cursor.Close(ctx)

       var logs []*LogEntry
       for cursor.Next(ctx) {
           var item LogEntry
           err := cursor.Decode(&item)
           if err != nil {
               log.Println("Error decoding log into slice:", err)
               return nil, err
           }
           logs = append(logs, &item)
       }

       return logs, nil
   }
   ```

   - **Получение контекста:** Используем контекст с тайм-аутом на 15 секунд для выполнения операции.
   - **Сортировка данных:** Опции поиска устанавливают сортировку по дате создания документа (`created_at`) в порядке убывания.
   - **Обработка курсора:** Мы перебираем результаты, используя курсор, и декодируем каждую запись в срез `logs`.

#### Заключение

Теперь у вас есть базовые знания о том, как подключаться к MongoDB, работать с коллекциями и моделями данных на языке Go. В следующем шаге вы можете расширить функциональность, добавив больше моделей данных и операций, таких как обновление и удаление записей, а также интеграцию с внешними сервисами для обработки и анализа данных.
