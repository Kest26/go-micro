### Подключение и наполнение базы данных для микросервиса аутентификации

#### 1. Подключение микросервиса аутентификации к базе данных

На данном этапе мы успешно подключили наш микросервис аутентификации к базе данных. Соединение прошло успешно, и микросервис смог отправить и получить ответ от базы данных (ping). Однако для полноценной работы приложения база данных должна содержать необходимые данные.

#### 2. Наполнение базы данных

Для этого лекции в ресурсах курса содержится архивный файл `users.sql.zip`. Его необходимо скачать, разархивировать и открыть в любом текстовом редакторе. Внутри файла находятся SQL-команды, которые выполняют следующие действия:
- Создают последовательность (`sequence`) для автоматического увеличения значений ID пользователей.
- Создают таблицу `users`.
- Добавляют одну запись в таблицу с данными пользователя `admin@example.com`, где пароль зашифрован (хэширован).

Примечание: Пароль пользователя — это слово "verysecret" (все буквы в нижнем регистре и без пробелов).

#### 3. Подключение к базе данных через клиент Postgres

Для взаимодействия с базой данных рекомендуется использовать клиент для Postgres. Если у вас нет установленного клиента, можно воспользоваться программой **BKeeper Studio**. Ссылка на скачивание предоставлена в ресурсах курса. Рекомендуется загрузить бесплатную версию Community Edition, которая поддерживается на всех основных операционных системах (Mac, Linux, Windows).

После установки и запуска BKeeper Studio, необходимо выполнить следующие действия:
1. Убедитесь, что ваш Docker Compose работает, чтобы база данных была доступна для подключения.
2. Настройте новое подключение к базе данных, выбрав тип `Postgres`. Параметры подключения:
   - Хост: `localhost`
   - Порт: `5432` (но мы используем `5433`, т.к. порт `5432` оказался занят)
   - Пользователь: `Postgres` (указано в файле Docker Compose)
   - Пароль: `password`
   - База данных по умолчанию: `users`
3. Проверьте подключение, и если все прошло успешно, нажмите "Connect".


#### 4. Импорт данных в базу

Оставьте открытым окно с запросом в BKeeper Studio и перейдите обратно в текстовый редактор, где открыт файл `users.sql`. Выделите весь текст и скопируйте его. 

```sql
--
-- Name: user_id_seq; Type: SEQUENCE; Schema: public; Owner: postgres
--

CREATE SEQUENCE public.user_id_seq
    START WITH 1
    INCREMENT BY 1
    NO MINVALUE
    NO MAXVALUE
    CACHE 1;


ALTER TABLE public.user_id_seq OWNER TO postgres;

SET default_tablespace = '';

SET default_table_access_method = heap;

--
-- Name: users; Type: TABLE; Schema: public; Owner: postgres
--

CREATE TABLE public.users (
    id integer DEFAULT nextval('public.user_id_seq'::regclass) NOT NULL,
    email character varying(255),
    first_name character varying(255),
    last_name character varying(255),
    password character varying(60),
    user_active integer DEFAULT 0,
    created_at timestamp without time zone,
    updated_at timestamp without time zone
);


ALTER TABLE public.users OWNER TO postgres;

--
-- Name: user_id_seq; Type: SEQUENCE SET; Schema: public; Owner: postgres
--

SELECT pg_catalog.setval('public.user_id_seq', 1, true);


--
-- Name: users users_pkey; Type: CONSTRAINT; Schema: public; Owner: postgres
--

ALTER TABLE ONLY public.users
    ADD CONSTRAINT users_pkey PRIMARY KEY (id);


INSERT INTO "public"."users"("email","first_name","last_name","password","user_active","created_at","updated_at")
VALUES
(E'admin@example.com',E'Admin',E'User',E'$2a$12$1zGLuYDDNvATh4RA4avbKuheAMpb1svexSzrQm7up.bnpwQHs0jNe',1,E'2022-03-14 00:00:00',E'2022-03-14 00:00:00');


```

Затем вернитесь в BKeeper Studio и вставьте скопированный SQL-код в окно запроса. Выберите все и выполните запрос.

После выполнения запроса в базе данных будет создана таблица `users`, и она будет содержать одну запись с данными пользователя `admin@example.com`. Теперь у нас есть информация в базе данных, с которой мы можем работать.

#### 5. Следующие шаги

На следующем занятии мы начнем процесс аутентификации, который будет включать взаимодействие между брокером и конечным пользователем на основе данных, содержащихся в нашей базе данных.
