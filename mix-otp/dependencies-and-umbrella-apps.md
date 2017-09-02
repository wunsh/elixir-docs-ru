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

## Зонтичные проекты

Давайте начнём новый проект, запустив `mix new`. Назовём этот проект `kv_umbrella`, а также добавим опцию `--umbrella` при создании. Не создавайте этот новый проект внутри существующего проекта `kv`!

```bash
$ mix new kv_umbrella --umbrella
* creating .gitignore
* creating README.md
* creating mix.exs
* creating apps
* creating config
* creating config/config.exs
```

Исходя из выведенной информации, мы можем увидеть, что сгенерировано намного меньше файлов. Сгенерированный `mix.exs` также отличается от стандартного. Давайте взглянем на него (коммантарии удалены):

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

Отличие от предыдущего проекта в строке `apps_path: "apps"` в определении проекта. Это значит, что проект будет вести себя, как зонтичный. Такие проекты не не содержат ни исходников, ни тестов, но они могут иметь свои зависимости. Каждое дочернее приложение должно быть определено внутри директории `apps`.

Давайте перейдём в директорию `apps` и начнём создавать `kv_server`. В этот раз мы передадим флаг `--sup`, который укажет Mix сгенерировать дерево супервизора автоматически, вместо того, чтобы создавать его руками, как мы делали в предыдущих главах:

```bash
$ cd kv_umbrella/apps
$ mix new kv_server --module KVServer --sup
```

Сгенерированные файлы идентичны сгенерированным для `kv`, но с некоторыми отличиями. Откройте `mix.exs`:

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

Первое, что можно заметить, Mix автоматически определил структуру зонтичного проекта, т.к. мы сгенерировали этот проект внутри `kv_umbrella/apps`, и добавил следующие строки:

```elixir
build_path: "../../_build",
config_path: "../../config/config.exs",
deps_path: "../../deps",
lockfile: "../../mix.lock",
```

Эти опции означают, что все зависимости будут взяты из `kv_umbrella/deps`, будут находиться в одной сборке, иметь общую конфигурацию и `lock` файлы. При этом зависимости будут загружены и скомпилированы один раз и для всего зонтичного проекта, а не для каждого приложения.

Второе изменение в функции `application` внутри `mix.exs`:

```elixir
def application do
  [
    extra_applications: [:logger],
    mod: {KVServer.Application, []}
  ]
end
```

Т.к. мы передали флаг `--sup`, Mix автоматически добавил `mod: {KVServer.Application, []}`, определяющий, что `KVServer.Application` - модуль обратного вызова приложения. `KVServer.Application` запустит дерево супервизора нашего приложения.

А теперь откроем `lib/kv_server/application.ex`:

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

Обратите внимание, что тут определена функция обратного вызова приложения, `start/2`, и вместо определения супервизора с именем `KVServer.Supervisor`, который использует модуль `Supervisor`, супервизор удобно определён в списке атрибутов! Вы можете прочитать больше о таких супервизорах в [документации модуля `Supervisor`](https://hexdocs.pm/elixir/Supervisor.html).

Мы уже можем попробовать наше первое приложение внутри проекта. Можно запустить тесты в директории `apps/kv_server`, но это будет не так круто, как перейти в корневую директорию зонтичного проекта и запустить `mix test`:

```bash
$ mix test
```

И это сработает!

Т.к. мы хотим, чтобы `kv_server` периодически использовал функциональность из приложения `kv`, нужно добавить его в зависимости нашего приложения.

## Зависимости внутри зонтичного проекта

Mix предоставляет лёгкий механизм для указания зависимостей одного приложения-потомка от другого. Откройте `apps/kv_server/mix.exs` и измените функцию `deps/0` на следующую:

```elixir
defp deps do
  [{:kv, in_umbrella: true}]
end
```

Строка выше делает `:kv` доступным как зависимость внутри `:kv_server`, и автоматически запускает приложение `:kv` до запуска сервера.

Наконец, скопируйте приложение `kv`, которое мы недавно создали, в директорию `apps` нашего нового зонтичного проекта. Конечная структура директорий должна соответствовать структуре, которую мы упоминали ранее:

    + kv_umbrella
      + apps
        + kv
        + kv_server

We now need to modify `apps/kv/mix.exs` to contain the umbrella entries we have seen in `apps/kv_server/mix.exs`. Open up `apps/kv/mix.exs` and add to the `project` function:

А теперь нам нужно изменить `apps/kv/mix.exs`, чтобы он содержал строки, специфичные для зонтичного приложения, которые мы видели в `apps/kv_server/mix.exs`. Откройте `apps/kv/mix.exs` и добавьте в функцию `project` следующее:

```elixir
build_path: "../../_build",
config_path: "../../config/config.exs",
deps_path: "../../deps",
lockfile: "../../mix.lock",
```

Теперь вы можете запускать тесты для обоих проектов из корневой директории зонта, запустив `mix test`. Супер!

Помните, что зонтичные проекты помогают удобно организовать ваши приложения и управлять ими. Приложения внутри директории `apps` остаются отдельными друг от труга. Зависимости между ними должны быть явно указаны. Это позволяет разрабатывать их вместе, но компилировать, тестировать и деплоить независимо, если это необходимо.

## Summing up

In this chapter, we have learned more about Mix dependencies and umbrella projects. While we may run `kv` without a server, our `kv_server` depends directly on `kv`. By breaking them into separate applications, we gain more control in how they are developed and tested.

When using umbrella applications, it is important to have a clear boundary between them. Our upcoming `kv_server` must only access public APIs defined in `kv`. Think of your umbrella apps as any other dependency or even Elixir itself: you can only access what is public and documented. Reaching into private functionality in your dependencies is a poor practice that will eventually cause your code to break when a new version is up.

Umbrella applications can also be used as a stepping stone for eventually extracting an application from your codebase. For example, imagine a web application that has to send "push notifications" to its users. The whole "push notifications system" can be developed as an umbrella application, with its own supervision tree and APIs. If you ever run into a situation where another project needs the push notifications system, extraction should be straightforward as long as the web application respects the push notification API boundary. Regardless if it happens in 2 weeks or in 3 years from development. Once extracted, the push notifications system can be moved to a private git repository or a public hex.pm package.

Developers may also use umbrella applications to break large business domains apart. The caution here is to make sure the domains don't depend on each other (also known as cyclic dependencies). If you run into such situations, it means those applications are not as isolated from each other as you originally thought, and you have architectural and design issues to solve. Overall, umbrella applications do not magically improve the design of your code. They can, however, help enforce boundaries when the code is well designed.

Finally, keep in mind that applications in an umbrella project all share the same configurations and dependencies. If two applications in your umbrella need to configure the same dependency in drastically different ways or even use different versions, such is impossible in umbrellas, and those apps likely need to be moved to separate projects.

With our umbrella project up and running, it is time to start writing our server.

