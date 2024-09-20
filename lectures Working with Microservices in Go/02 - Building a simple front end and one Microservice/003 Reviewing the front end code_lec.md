Конечно, давайте перескажем лекцию, сохранив все важные детали и подробно объяснив работу кода.

---

Мы будем активно использовать этот фронтенд-код на протяжении всего курса, поэтому важно его рассмотреть. Код очень простой.

Откройте рабочее пространство в Visual Studio Code и убедитесь, что папка `front-end` уже добавлена. Внутри этой папки находятся директория `cmd`, поддиректория `templates` и один файл Go под названием `main.go`.

Давайте рассмотрим содержимое файла `main.go`.

### main.go

```go
package main

import (
	"fmt"
	"html/template"
	"log"
	"net/http"
)

func main() {
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		render(w, "test.page.gohtml")
	})

	fmt.Println("Starting front end service on port 80")
	err := http.ListenAndServe(":80", nil)
	if err != nil {
		log.Panic(err)
	}
}
```

#### Функция main

В функции `main` мы создаем HTTP-обработчик для корневого маршрута (`/`). Когда кто-то переходит на корневой уровень нашего приложения, вызывается анонимная функция, которая в свою очередь вызывает функцию `render`, передавая ей ответный объект (`w`) и имя шаблона `test.page.gohtml`.

Затем мы запускаем HTTP-сервер на порту 80 с помощью `http.ListenAndServe`. Если при запуске сервера возникает ошибка, она логируется и программа завершает работу.

### Функция render

```go
func render(w http.ResponseWriter, t string) {
	partials := []string{
		"./cmd/web/templates/base.layout.gohtml",
		"./cmd/web/templates/header.partial.gohtml",
		"./cmd/web/templates/footer.partial.gohtml",
	}

	var templateSlice []string
	templateSlice = append(templateSlice, fmt.Sprintf("./cmd/web/templates/%s", t))

	for _, x := range partials {
		templateSlice = append(templateSlice, x)
	}

	tmpl, err := template.ParseFiles(templateSlice...)
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}

	if err := tmpl.Execute(w, nil); err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
	}
}
```

#### Шаблоны и функция render

Функция `render` отвечает за рендеринг HTML-шаблонов. Вначале мы создаем срез строк `partials`, который содержит пути к файлам шаблонов: базовый шаблон (`base.layout.gohtml`), заголовок (`header.partial.gohtml`) и подвал (`footer.partial.gohtml`).

Затем мы создаем пустой срез строк `templateSlice` и добавляем в него путь к основному шаблону, переданному в функцию в качестве аргумента `t`. В данном случае это `test.page.gohtml`.

После этого мы добавляем в `templateSlice` все шаблоны из среза `partials`.

Затем мы используем метод `template.ParseFiles`, чтобы распарсить все шаблоны из `templateSlice`. Если возникает ошибка, мы отправляем ответ с кодом состояния HTTP 500.

Если парсинг прошел успешно, мы выполняем шаблон с помощью метода `tmpl.Execute`. Если при выполнении возникает ошибка, мы также отправляем ответ с кодом состояния HTTP 500.

### Шаблоны

#### base.layout.gohtml

```html
{{define "base" }}
<!doctype html>
<html lang="en">

{{template "header" .}}

<body>

{{block "content" .}}
{{end}}

{{block "js" .}}
{{end}}

{{template "footer" .}}

</body>
</html>
{{end}}
```

Этот файл определяет базовую структуру HTML-документа, включая заголовок и подвал, а также блоки для основного контента и JavaScript-кода.

#### header.partial.gohtml

```html
{{define "header"}}
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, user-scalable=no, initial-scale=1.0, maximum-scale=1.0, minimum-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Microservices in Go</title>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.1.3/dist/css/bootstrap.min.css" rel="stylesheet" integrity="sha384-1BmE4kWBq78iYhFldvKuhfTAU6auU8tT94WrHftjDbrCEXSU1oBoqyl2QvZ6jIW3" crossorigin="anonymous">
</head>
{{end}}
```

Этот файл определяет заголовок HTML-документа, включая мета-теги и подключение CSS-фреймворка Bootstrap.

#### footer.partial.gohtml

```html
{{define "footer"}}
<div class="container">
    <div class="row">
        <div class="col text-center">
            <hr>
            <small class="text-muted">Copyright &copy; GoCode.ca</small>
        </div>
    </div>
</div>
{{end}}
```

Этот файл определяет подвал HTML-документа, включающий контейнер с текстом "Copyright &copy; GoCode.ca".

#### test.page.gohtml

```html
{{template "base" .}}

{{define "content"}}
<div class="container">
    <div class="row">
        <div class="col">
            <h1 class="mt-5">Test microservices</h1>
            <hr>
            <div id="output" class="mt-5" style="outline: 1px solid silver; padding: 2em;">
                <span class="text-muted">Output shows here...</span>
            </div>
        </div>
    </div>
    <div class="row">
        <div class="col">
            <h4 class="mt-5">Sent</h4>
            <div class="mt-1" style="outline: 1px solid silver; padding: 2em;">
                <pre id="payload"><span class="text-muted">Nothing sent yet...</span></pre>
            </div>
        </div>
        <div class="col">
            <h4 class="mt-5">Received</h4>
            <div class="mt-1" style="outline: 1px solid silver; padding: 2em;">
                <pre id="received"><span class="text-muted">Nothing received yet...</span></pre>
            </div>
        </div>
    </div>
</div>
{{end}}

{{define "js"}}
<script>
    // JavaScript code will go here
</script>
{{end}}
```

Этот файл использует базовый шаблон `base`, добавляя основной контент страницы. Внутри контейнера находится один ряд с заголовком "Test microservices" и двумя блоками для вывода отправленных и полученных данных.

### Запуск приложения

Чтобы запустить приложение, откройте терминал, перейдите в папку `front-end` и выполните команду `go run ./cmd/web`. Эта команда запустит HTTP-сервер на порту 80. Откройте веб-браузер и перейдите по адресу `http://localhost`. Вы увидите сгенерированную страницу.

### Подведение итогов

Мы рассмотрели структуру простого веб-приложения на Go, включающего базовый шаблон, заголовок, подвал и основной контент. В следующей лекции мы создадим простой микросервис брокера и проверим возможность соединения фронтенда с этим микросервисом.

---

Код из лекции:

`cmd/web/main.go`:

```go
package main

import (
	"fmt"
	"html/template"
	"log"
	"net/http"
)

func main() {
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		render(w, "test.page.gohtml")
	})

	fmt.Println("Starting front end service on port 80")
	err := http.ListenAndServe(":80", nil)
	if err != nil {
		log.Panic(err)
	}
}

func render(w http.ResponseWriter, t string) {
	partials := []string{
		"./cmd/web/templates/base.layout.gohtml",
		"./cmd/web/templates/header.partial.gohtml",
		"./cmd/web/templates/footer.partial.gohtml",
	}

	var templateSlice []string
	templateSlice = append(templateSlice, fmt.Sprintf("./cmd/web/templates/%s", t))

	for _, x := range partials {
		templateSlice = append(templateSlice, x)
	}

	tmpl, err := template.ParseFiles(templateSlice...)
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}

	if err := tmpl.Execute(w, nil); err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
	}
}
```