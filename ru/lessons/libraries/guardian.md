---
version: 1.0.0
layout: page
title: Guardian (Основы)
category: libraries
order: 1
lang: ru
---

[Guardian](https://github.com/ueberauth/guardian) &mdash; часто используемая библиотека, основанная на [JWT](https://jwt.io/) (JSON Web Token).

{% include toc.html %}

## JWT

Что такое JWT? JWT занимается созданием токенов для аутентификации. В то время как многие системы аутентификации предоставляют доступ только к идентификатору ресурса, JWT позволяет хранить еще и другую информацию, такую как:

* Кем выдан токен
* Для кого выдан токен
* Какие системы должны использовать этот токен
* На какое время он выдан
* Каков тип токена
* Какие права есть у владельца данного токена

Это всё &mdash; простые примеры полей в JWT. Технология позволяет добавить дополнительную информацию, которой требует логика приложения. Главное &mdash; не забывайте что этот токен должен быть коротким, ведь он должен поместиться в HTTP заголовке.

В результате такого подхода JWT токены могут передаваться как полноценный целостный элемент данных идентификации пользователя.

### Когда их использовать

Токены JWT могут использоваться для аутентификации в любой части приложения:

* Одностраничные приложения
* Контроллеры (как с помощью браузерной сессии, так и api: с помощью HTTP заголовков)
* Phoenix Channels
* Запросы между сервисами
* Запросы между процессами
* Функциональность "запомнить меня"
* Другие интерфейсы - чистый TCP, UDP, CLI

Такие токены могут использоваться в приложении где угодно, когда нужно предоставить проверяемую аутентификцию.

### Нужно ли использовать базу данных?

JWT токены не нужно отслеживать с помощью базы данных. Достаточно знать, что они уже выписаны и использовать время жизни токена для контроля доступа. Часто встречаются случаи, когда нужно сделать запрос в базу для поиска/получения нужного ресурса, однако JWT это не требует.

Например, если нужно аутентифицировать подключение к UDP сокету с помощью JWT, можно обойтись без подключения к базе данных. Вся необходимая информация уже будет встроена в токен в момент его создания. Данные из токена можно использовать сразу после проверки подписи.

Однако, _можно_ использовать базу данных для отслеживания JWT токенов. Если воспользоваться этой возможностью, можно проверить, что токен не был отозван. Как вариант, можно использовать записи в базе данных для отзыва сразу всех токенов пользователя. Это довольно просто реализуется в Guardian с помощью [GuardianDb](https://github.com/hassox/guardian_db). GuardianDb использует интерфейсы Guardian для проведения валидации, сохранения и удаления из базы данных. Мы дойдем до этой темы позже.

## Настройка

В Guardian довольно много различных настроек. Мы рассмотрим их позже, но начнем с самого простого варианта.

### Минимальная настройка

Для начала работы с Guardian нужно сделать несколько вещей.

#### Настройка

`mix.exs`

```elixir
def application do
  [
    mod: {MyApp, []},
    applications: [:guardian, ...]
  ]
end

def deps do
  [
    {guardian: "~> x.x"},
    ...
  ]
end
```

`config/config.ex`

```elixir
config :guardian, Guardian,
  issuer: "MyAppId",
  secret_key: Mix.env, # в каждом окружении стоит переопределять этот ключ
  serializer: MyApp.GuardianSerializer
```

Это минимальный набор информации, который нужно предоставить Guardian для функционирования. Не стоит встраивать секретный ключ напрямую в основную конфигурацию приложения. Вместо этого у каждого окружения должен быть свой ключ. Хорошей практикой является использовать название Mix окружения в качестве значения ключа для dev и test окружений. В то же время, staging и production окружения должны использовать сложные секретные ключи (например, сгенерированный с помощью `mix phoenix.gen.secret`).

`lib/my_app/guardian_serializer.ex`

```elixir
defmodule MyApp.GuardianSerializer do
  @behaviour Guardian.Serializer

  alias MyApp.Repo
  alias MyApp.User

  def for_token(user = %User{}), do: { :ok, "User:#{user.id}" }
  def for_token(_), do: { :error, "Неизвестный ресурс" }

  def from_token("User:" <> id), do: { :ok, Repo.get(User, id) }
  def from_token(_), do: { :error, "Неизвестный ресурс" }
end
```

Сериализатор отвечает за определение нужного ресурса на основе данных из токена. Это может быть поиск в базе данных, внешнем API или даже простая строка. Он также отвечает за обратный процесс превращения объекта пользователя в сериализованный вариант.

Это всё, что нужно для минимальной настройки. Есть еще много других настроек, но для начала этого достаточно.

#### Использование в приложении

Теперь, когда всё настроено, нам нужно интегрировать Guardian в приложение. Так как это минималистичный вариант, давайте сначала рассмотрим его использование в контексте HTTP запросов.

## HTTP запросы

Guardian предоставляет различные Plug для интеграции в HTTP запросы. О Plug можно почитать в [другом уроке](../specifics/plug/). Guardian не требует Phoenix, с ним легче всего будет привести следующие примеры, потому мы будем его использовать.

Самый легкий способ интеграции &mdash; роутер. Так как HTTP интеграции Guardian основаны на plug-ах, их можно использовать в любом совместимом инструментарии.

Основная логика Guardian plug выглядит так:

1. Найти токен внутри запроса и проверить его: `Verify...` plug-и.
2. Опционально загрузить ресурсы, которые указаны в токене: plug `LoadResource`.
3. Проверить, что найден пользователь. Иначе &mdash; запретить доступ: plug `EnsureAuthenticated`.

Для большей гибкости Guardian реализует все эти фазы отдельными plug-ами.

Давайте воспользуемся этими знаниями:

```elixir
pipeline :maybe_browser_auth do
  plug Guardian.Plug.VerifySession
  plug Guardian.Plug.VerifyHeader, realm: "Bearer"
  plug Guardian.Plug.LoadResource
end

pipeline :ensure_authed_access do
  plug Guardian.Plug.EnsureAuthenticated, %{"typ" => "access", handler: MyApp.HttpErrorHandler}
end
```

Эти pipeline могут быть использованы, чтобы собрать из них реализацию различных требований к аутентификации. Первый вариант сначала пытается найти токен в сессии, потом &mdash; в заголовках запроса, и, если находит, загружает ресурс.

Второй &mdash; требует наличия корректного проверенного токена типa `access`. Для использования их нужно добавить в роутер:

```elixir
scope "/", MyApp do
  pipe_through [:browser, :maybe_browser_auth]

  get "/login", LoginController, :new
  post "/login", LoginController, :create
  delete "/login", LoginController, :delete
end

scope "/", MyApp do
  pipe_through [:browser, :maybe_browser_auth, :ensure_authed_access]

  resource "/protected/things", ProtectedController
end
```

Путь `login` в примере выше будет с аутентифицированными пользователем (если таковой есть). Во втором же примере мы проверяем, что токен присутствует и правилен. Нет никаких требований по обязательному наличию их в процессе обработки запросов, можно добавить их только в тех местах, где это необходимо.

На данный момент всё ещё не хватает одной части &mdash; обработчика ошибок, возникающих в `EnsureAuthenticated`. Это очень простой модуль, у которого должны быть реализованы две функции:

* `unauthenticated/2`
* `unauthorized/2`

Обе эти функции принимают в качестве параметров Plug.Conn и параметры запроса. Для этого можно использовать даже Phoenix контроллер.

#### Контроллер

В контроллере есть несколько вариантов получения доступа к текущему пользователю. Начнем с самого простого:

```elixir
defmodule MyApp.MyController do
  use MyApp.Web, :controller
  use Guardian.Phoenix.Controller

  def some_action(conn, params, user, claims) do
    # Здесь может быть какой-то обработчик
  end
end
```

После подключения модуля `Guardian.Phoenix.Controller` обработчики будут получать два дополнительных параметра, по которым можно делать сопоставление с образцом. Стоит заметить, что если не использовался `EnsureAuthenticated`, то вместо пользователя может быть `nil`.

Второй, более гибкий/очевидный вариант &mdash; использовать специальные функции самого `Guardian`.

```elixir
defmodule MyApp.MyController do
  use MyApp.Web, :controller

  def some_action(conn, params) do
    if Guardian.Plug.authenticated?(conn) do
      user = Guardian.Plug.current_resource(conn)
    else
      # Пользователь не найден
    end
  end
end
```

#### Авторизация/Выход

Реализация входа и выхода с использованием браузерной сессии довольно проста. В контроллере входа:

```elixir
def create(conn, params) do
  case find_the_user_and_verify_them_from_params(params) do
    {:ok, user} ->
      conn
      |> Guardian.Plug.sign_in(user, :access) # С указанием типа токена `access`. Другие типы также могут быть использованы (например, `:refresh`)
      |> respond_somehow()
    {:error, reason} ->
      # Обработка ситуации, когда предоставлены неверные данные
  end
end

def delete(conn, params) do
  conn
  |> Guardian.Plug.sign_out()
  |> respond_somehow()
end
```

Когда речь идет об авторизации в API &mdash; ситуация слегка отличается, ведь тогда нет сессии и нужно передать токен клиенту.
Для входа в API скорее всего будет использоваться заголовок `Authorization` в запросах к приложению.

```elixir
def create(conn, params) do
  case find_the_user_and_verify_them_from_params(params) do
    {:ok, user} ->
      {:ok, jwt, _claims} = Guardian.encode_and_sign(user, :access)
      conn
      |> respond_somehow({token: jwt})
    {:error, reason} ->
      # Обработка ситуации, когда предоставлены неверные данные
  end
end

def delete(conn, params) do
  jwt = Guardian.Plug.current_token(conn)
  Guardian.revoke!(jwt)
  respond_somehow(conn)
end
```

Реализация логина через браузерную сессию самостоятельно вызывает `encode_and_sign` внутри имплементации `Guardian.Plug.sign_in`. Потому оба приведённых примера эквивалентны.