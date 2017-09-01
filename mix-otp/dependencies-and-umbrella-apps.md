---
title: Зависимости и зонтичные проекты
---

# {{ page.title }}

В этой главе мы обсудим, как управлять зависимостями с помощью Mix.

Наше приложение `kv` закончино, и значит пришло время сделать сервер, который будет обрабатывать закросы, которые мы объявили в первой главе:

```
CREATE shopping
OK

PUT shopping milk 1
OK

PUT shopping eggs 3
OK

GET shopping milk
1
OK

DELETE shopping eggs
OK
```

Однако, вместо добавления ещё большего количества кода в приложение `kv`, мы сделаем TCP сервер отдельным приложением, которое будет клиентом приложения `kv`. Т.к. весь рантайм и экосистема Эликсира основана на приложениях, есть смысл разделять наши проекты в приложения меньшего размера, которые работают вместе, вместо создания одного большого монолитного приложения.

Перед тем, как создать наше новое приложение, мы должны обсудить, как Mix разрешает зависимости. На практике существует два типа зависимостей, с которыми мы обычно работаем: внутренние и внешние зависимости. Mix поддерживает механизмы для работы с обоими.

## Внешние зависимости

Внешними называют зависимости, которые не относятся к вашей бизнес-логике. Например, если вам нужен HTTP API для распределённого приложения KV, вы можете использовать проект [Plug](https://github.com/elixir-lang/plug) как внешнюю зависимость.

Установка внешних зависимостей предельна проста. Как правило, мы используем [Пакетный Менеджер Hex](https://hex.pm), перечисляя зависимости в функции `deps` в нашем файле `mix.exs`:

```elixir
def deps do
  [{:plug, "~> 1.0"}]
end
```

Эта зависимость ссылается на последнюю версию Plug в серии версий 1.x.x, которая известна Hex. Для указания последней версии используется `~>` перед номером минимальной версии. Больше информации о версиях можно получить в [документации к модулю Version](https://hexdocs.pm/elixir/Version.html).

Обычно стабильные релизы загружены в Hex. Если вы хотите добавить внешнюю зависимость, которая ещё находится в разработке, Mix позволяет указать git зависимост:

```elixir
def deps do
  [{:plug, git: "git://github.com/elixir-lang/plug.git"}]
end
```

Вы можете заметить, что при добавлении зависимости в ваш проект Mix генерирует файл `mix.lock`, который гарантирует *повторяемость сборок*. Файл блокировки должен быть включен в вашу систему контроля версии, чтобы гарантировать для всех пользователей проекта использование тех же версий зависимостей, что и у вас.

Mix предоставляет много функций для работы с зависимостями, которые можно увидеть, вызвав `mix help`:

```bash
$ mix help
mix deps              # Lists dependencies and their status
mix deps.clean        # Deletes the given dependencies' files
mix deps.compile      # Compiles dependencies
mix deps.get          # Gets all out of date dependencies
mix deps.tree         # Prints the dependency tree
mix deps.unlock       # Unlocks the given dependencies
mix deps.update       # Updates the given dependencies
```

Чаще всего используются функции `mix deps.get` и `mix deps.update`. Однажды загруженные, зависимости будут автоматически скомпилированы. Вы можете узнать больше про `deps`, если вызовите `mix help deps`, а также из [документации к модулю Mix.Tasks.Deps](https://hexdocs.pm/mix/Mix.Tasks.Deps.html).

## Внутренние зависимости

Специфичные для вашего проекта зависимости называют внутренними. Они обычно не имеют смысла вне вашего проекта/компании/организации. Как правило, вы хотите сохранить их приватными, ввиду технических, экономических или бизнес причин.

Если у вас есть внутренняя зависимость, Mix поддерживает два метода работы с ними: репозитории git и зонтичные проекты.

Например, если вы храните проект `kv` в git репозиторие, вам нужно указать его в списке зависимостей, чтобы начать его использовать:

```elixir
def deps do
  [{:kv, git: "https://github.com/YOUR_ACCOUNT/kv.git"}]
end
```

Если же репозиторий приватный, вам понадобится приватный URL `git@github.com:YOUR_ACCOUNT/kv.git`. В этом случае Mix сможет получать доступ к репозиторию, пока у вас есть права доступа.

Использование git для внутренних зависимостей в Эликсире не лучший вариант. Помните, что экосистема Эликсира следует концепции приложении. Поэтому мы надеемся, что вы часто будете разделять свой код на приложения, которые могут быть логически организованны, даже в рамках одного проекта.

Однако, если каждое приложение будет иметь свой репозиторий, ваш проект станет очень сложным в поддержке, вы будете тратить много времени на управление этими репозиториями вместо написания кода.

По этой причине Mix поддерживает "зонтичные проекты". Зонтичные проекты используются для разработки приложений, которые работают вместе и имеют четкие границы внутри одного репозитория. Именно этот стиль мы изучим в следующих разделах.

Давайте создадим новый проект Mix. Мы оригинально назовём его `kv_umbrella`, и этот проект будет включать в себя уже созданное приложение `kv` и новое, `kv_server`. Структура директорий должна выглядить следующим образом:

    + kv_umbrella
      + apps
        + kv
        + kv_server

Интересно, что для этого подхода в Mix есть много удобных возможностей, например, возможность компилировать и тестировать все приложения внутри `apps` одной командой. Однако, хотя они и перечислены вместе внутри `apps`, это всё ещё отдельные, изолированные друг от друга приложения, поэтому мы можем собирать, тестировать и деплоить каждое приложение отдельно, если это необходимо.

Что ж, начнём!

## Umbrella projects

Let's start a new project using `mix new`. This new project will be named `kv_umbrella` and we need to pass the `--umbrella` option when creating it. Do not create this new project inside the existing `kv` project!

```bash
$ mix new kv_umbrella --umbrella
* creating .gitignore
* creating README.md
* creating mix.exs
* creating apps
* creating config
* creating config/config.exs
```

From the printed information, we can see far fewer files are generated. The generated `mix.exs` file is different too. Let's take a look (comments have been removed):

```elixir
defmodule KvUmbrella.Mixfile do
  use Mix.Project

  def project do
    [
      apps_path: "apps",
      start_permanent: Mix.env == :prod,
      deps: deps
    ]
  end

  defp deps do
    []
  end
end
```

What makes this project different from the previous one is the `apps_path: "apps"` entry in the project definition. This means this project will act as an umbrella. Such projects do not have source files nor tests, although they can have their own dependencies. Each child application must be defined inside the `apps` directory.

Let's move inside the apps directory and start building `kv_server`. This time, we are going to pass the `--sup` flag, which will tell Mix to generate a supervision tree automatically for us, instead of building one manually as we did in previous chapters:

```bash
$ cd kv_umbrella/apps
$ mix new kv_server --module KVServer --sup
```

The generated files are similar to the ones we first generated for `kv`, with a few differences. Let's open up `mix.exs`:

```elixir
defmodule KVServer.Mixfile do
  use Mix.Project

  def project do
    [
      app: :kv_server,
      version: "0.1.0",
      build_path: "../../_build",
      config_path: "../../config/config.exs",
      deps_path: "../../deps",
      lockfile: "../../mix.lock",
      elixir: "~> 1.6-dev",
      start_permanent: Mix.env == :prod,
      deps: deps()
    ]
  end

  # Run "mix help compile.app" to learn about applications.
  def application do
    [
      extra_applications: [:logger],
      mod: {KVServer.Application, []}
    ]
  end

  # Run "mix help deps" to learn about dependencies.
  defp deps do
    [
      # {:dep_from_hexpm, "~> 0.3.0"},
      # {:dep_from_git, git: "https://github.com/elixir-lang/my_dep.git", tag: "0.1.0"},
      # {:sibling_app_in_umbrella, in_umbrella: true},
    ]
  end
end
```

First of all, since we generated this project inside `kv_umbrella/apps`, Mix automatically detected the umbrella structure and added four lines to the project definition:

```elixir
build_path: "../../_build",
config_path: "../../config/config.exs",
deps_path: "../../deps",
lockfile: "../../mix.lock",
```

Those options mean all dependencies will be checked out to `kv_umbrella/deps`, and they will share the same build, config and lock files. This ensures dependencies will be fetched and compiled once for the whole umbrella structure, instead of once per umbrella application.

The second change is in the `application` function inside `mix.exs`:

```elixir
def application do
  [
    extra_applications: [:logger],
    mod: {KVServer.Application, []}
  ]
end
```

Because we passed the `--sup` flag, Mix automatically added `mod: {KVServer.Application, []}`, specifying that `KVServer.Application` is our application callback module. `KVServer.Application` will start our application supervision tree.

In fact, let's open up `lib/kv_server/application.ex`:

```elixir
defmodule KVServer.Application do
  # See https://hexdocs.pm/elixir/Application.html
  # for more information on OTP Applications
  @moduledoc false

  use Application

  def start(_type, _args) do
    # List all child processes to be supervised
    children = [
      # Starts a worker by calling: KVServer.Worker.start_link(arg)
      # {KVServer.Worker, arg},
    ]

    # See https://hexdocs.pm/elixir/Supervisor.html
    # for other strategies and supported options
    opts = [strategy: :one_for_one, name: KVServer.Supervisor]
    Supervisor.start_link(children, opts)
  end
end
```

Notice that it defines the application callback function, `start/2`, and instead of defining a supervisor named `KVServer.Supervisor` that uses the `Supervisor` module, it conveniently defined the supervisor inline! You can read more about such supervisors by reading [the Supervisor module documentation](https://hexdocs.pm/elixir/Supervisor.html).

We can already try out our first umbrella child. We could run tests inside the `apps/kv_server` directory, but that wouldn't be much fun. Instead, go to the root of the umbrella project and run `mix test`:

```bash
$ mix test
```

And it works!

Since we want `kv_server` to eventually use the functionality we defined in `kv`, we need to add `kv` as a dependency to our application.

## In umbrella dependencies

Mix supports an easy mechanism to make one umbrella child depend on another. Open up `apps/kv_server/mix.exs` and change the `deps/0` function to the following:

```elixir
defp deps do
  [{:kv, in_umbrella: true}]
end
```

The line above makes `:kv` available as a dependency inside `:kv_server` and automatically starts the `:kv` application before the server starts.

Finally, copy the `kv` application we have built so far to the `apps` directory in our new umbrella project. The final directory structure should match the structure we mentioned earlier:

    + kv_umbrella
      + apps
        + kv
        + kv_server

We now need to modify `apps/kv/mix.exs` to contain the umbrella entries we have seen in `apps/kv_server/mix.exs`. Open up `apps/kv/mix.exs` and add to the `project` function:

```elixir
build_path: "../../_build",
config_path: "../../config/config.exs",
deps_path: "../../deps",
lockfile: "../../mix.lock",
```

Now you can run tests for both projects from the umbrella root with `mix test`. Sweet!

Remember that umbrella projects are a convenience to help you organize and manage your applications. Applications inside the `apps` directory are still decoupled from each other. Dependencies between them must be explicitly listed. This allows them to be developed together, but compiled, tested and deployed independently if desired.

## Summing up

In this chapter, we have learned more about Mix dependencies and umbrella projects. While we may run `kv` without a server, our `kv_server` depends directly on `kv`. By breaking them into separate applications, we gain more control in how they are developed and tested.

When using umbrella applications, it is important to have a clear boundary between them. Our upcoming `kv_server` must only access public APIs defined in `kv`. Think of your umbrella apps as any other dependency or even Elixir itself: you can only access what is public and documented. Reaching into private functionality in your dependencies is a poor practice that will eventually cause your code to break when a new version is up.

Umbrella applications can also be used as a stepping stone for eventually extracting an application from your codebase. For example, imagine a web application that has to send "push notifications" to its users. The whole "push notifications system" can be developed as an umbrella application, with its own supervision tree and APIs. If you ever run into a situation where another project needs the push notifications system, extraction should be straightforward as long as the web application respects the push notification API boundary. Regardless if it happens in 2 weeks or in 3 years from development. Once extracted, the push notifications system can be moved to a private git repository or a public hex.pm package.

Developers may also use umbrella applications to break large business domains apart. The caution here is to make sure the domains don't depend on each other (also known as cyclic dependencies). If you run into such situations, it means those applications are not as isolated from each other as you originally thought, and you have architectural and design issues to solve. Overall, umbrella applications do not magically improve the design of your code. They can, however, help enforce boundaries when the code is well designed.

Finally, keep in mind that applications in an umbrella project all share the same configurations and dependencies. If two applications in your umbrella need to configure the same dependency in drastically different ways or even use different versions, such is impossible in umbrellas, and those apps likely need to be moved to separate projects.

With our umbrella project up and running, it is time to start writing our server.

