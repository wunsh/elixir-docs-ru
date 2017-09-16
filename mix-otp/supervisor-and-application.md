---
title: Супервизор и Приложение
---

# {{ page.title }}

В нашем приложении теперь есть реестр, который может работать с дюжинами, если не с сотнями корзин. Мы можем думать, что наша реализация достаточно хороша, но ПО никогда не бывает без багов, и падения будут случаться.

Когда это происходит, ваша первая реакция может быть: "добавлю rescue для обработки ошибок". Но в Эликсире мы избегаем "защитного" программирования с отловом ошибок. напротив, мы говорим "пускай падает". Если есть баг, который приводит к падению реестра, у нас нет повода волноваться, потому что мы сделаем супервизор, который запустит новую копию реестра.

В этой плаве мы изучим супервизоры и, также, приложения. Мы создадим не один, а два супревизора, и и используем их для наблюдения за нашими процессами.

## Наш первый супервизор

Создание супервизора не слишком отличается от создания GenServer. Мы определим модуль с именем `KV.Supervisor`, который будет использовать поведение [Supervisor](https://hexdocs.pm/elixir/Supervisor.html), внутри файла `lib/kv/supervisor.ex`:

```elixir
defmodule KV.Supervisor do
  use Supervisor

  def start_link(opts) do
    Supervisor.start_link(__MODULE__, :ok, opts)
  end

  def init(:ok) do
    children = [
      KV.Registry
    ]

    Supervisor.init(children, strategy: :one_for_one)
  end
end
```

Пока наш супервизор имеет единственного потомка: `KV.Registry`. Когда мы определим несколько потомков, мы будем вызывать `Supervisor.init/2`, передавая потомков и стратегию надзора за ними.

Стратегия надзора задаёт поведение в случае падения одного из потомков. `:one_for_one` значит, что при падении потомка, только он один будет перезапущен. Пока у нас только один потомок, это то, что нужно. Поведение `Supervisor` поддерживает много разных стратегий, и мы поговорим о них в этой главе.

После старта супервизор пройдёт по списку потомков и выполнит фукнцию `child_spec/1` для каждого модуля. Мы слышали о функции `child_spec/1` в главе "Агенты", когда вызывали `start_supervised(KV.Bucket)` без указания модуля.

Функция `child_spec/1` возвращает спецификацию потомка, которая объясняет, как запустить процесс, является ли он воркером или супервизором, является ли он временным или постоянным и т.д. Функция `child_spec/1` автоматически определяется, когда мы задаём `use Agent`, `use GenServer`, `use Supervisor` и т.д. Давайте попробуем это на практике, запустив `iex -S mix`:

```iex
iex(1)> KV.Registry.child_spec([])
%{
  id: KV.Registry,
  restart: :permanent,
  shutdown: 5000,
  start: {KV.Registry, :start_link, [[]]},
  type: :worker
}
```

Мы изучим другие детали по ходу этого руководства. Если вы хотите понять больше, загляните в раздел документации [Supervisor](https://hexdocs.pm/elixir/Supervisor.html).

Когда супрвизор получит все спецификации потомков, он запустит их один за другим, в порядке, в котором они определены, используя информацию по ключу `:start` в их спецификациях. В текущей спецификации он вызовет `KV.Registry.start_link([])`.

Пока `start_link/1` всегда получает пустой список опций. Самое время изменить это.

## Именование процессов

Наше приложение будет иметь много корзин, но только один реестр. Поэтому вместо передачи всюду PID реестра, мы можем дать ему имя и ссылаться на него по его имени.

Также, как вы помните, корзины создаются динамически основываясь на пользовательском вводе, поэтому не следует использовать атомы для управления корзинами. Но реестр будет один, запущенный супервизором после старта нашего приложения.

Давайте реализуем это. Немного изменим наше определение потомков из списка атомов в список кортежей:

```elixir
  def init(:ok) do
    children = [
      {KV.Registry, name: KV.Registry}
    ]

    Supervisor.init(children, strategy: :one_for_one)
  end
```

Разница в том, что вместо вызова `KV.Registry.start_link([])` супервизор будет вызывать `KV.Registry.start_link([name: KV.Registry])`. Если вы вернётесь к реализации `KV.Registry.start_link/1`, вы вспомните, что при этом будут переданы опции в GenServer

```elixir
  def start_link(opts) do
    GenServer.start_link(__MODULE__, :ok, opts)
  end
```

который зарегистрирует процесс с переданным именем.

Давайте попробуем всё это в `iex -S mix`:

```iex
iex> KV.Supervisor.start_link([])
{:ok, #PID<0.66.0>}
iex> KV.Registry.create(KV.Registry, "shopping")
:ok
iex> KV.Registry.lookup(KV.Registry, "shopping")
{:ok, #PID<0.70.0>}
```

When we started the supervisor, the registry was automatically started with the given name, allowing us to create buckets without the need to manually start it.

In practice, we rarely start the application supervisor manually. Instead, it is started as part of the application callback.

На практике редко запускают супервизор приложения вручную. Напротив, он стартует как часть обратного вызова в приложении.

## Понимание приложений

Мы всё время работали внутри приложения. Каждый раз, когда мы изменяли файл и запускали `mix compile`, мы могли увидеть сообщение `Generated kv app` в выводе компиляции.

Мы можем найти сгенерированный файл `.app` в `_build/dev/lib/kv/ebin/kv.app`. Давайте посмотрим на его содержимое:

```erlang
{application,kv,
             [{registered,[]},
              {description,"kv"},
              {applications,[kernel,stdlib,elixir,logger]},
              {vsn,"0.0.1"},
              {modules,['Elixir.KV','Elixir.KV.Bucket',
                        'Elixir.KV.Registry','Elixir.KV.Supervisor']}]}.
```

Этот файл содержит термы Эрланга (написан с использованием синтаксиса Эрланга). Даже если мы не знакомы с Эрлангом, достаточно просто предположить, что это определение приложения. Оно содержит версию нашего приложения (`version`), все определённые в нём модули, а также список приложений-зависимостей, например, `kernel` Эрланга, сам `elixir` и `logger`, который определён в списке `:extra_applications` внутри `mix.exs`.

Былобы очень неудобно обновлять этот файл вручную каждый раз, когда мы добавляем новый модуль в наше приложение. Поэтому Mix генерирует и поддерживает его актуальным за нас.

Мы также можем настроить генерируемый файл `.app`, изменив значения, возвращаемые `application/0` внутри нашего файла проекта `mix.exs`. Мы скоро сделаем наши первые изменения.

### Запуск приложений

Когда мы определяем файл `.app`, который является спецификацией приложения, мы можем запускать и останавливать приложение целиком. До сих пор мы не думали об этом по двум причинам:

1. Mix автоматически запускал приложение за нас

2. Даже если Mix не запускал наше приложение за нас, оно ничего не делало при запуске

В любом случае, давайте посмотрим, как Mix запускает приложение за нас. Откройте консоль проекта с помощью `iex -S mix` и попробуйте:

```iex
iex> Application.start(:kv)
{:error, {:already_started, :kv}}
```

Упс, оно уже запущено. Mix по умолчанию запускает всю иерархию приложений, объявленную в файле проекта `mix.exs` и то же происходит со всеми зависимостями, если они зависят от других приложений.

Мы можем с помощью опций запустить Mix с указанием не запускать приложение. Попробуйте ввести в консоль `iex -S mix run --no-start`:

```iex
iex> Application.start(:kv)
:ok
```

Мы можем остановить наше приложение `:kv` и, точно так же, приложение `:logger`, которое запустилось по умолчанию вместе с Эликсиром:

```iex
iex> Application.stop(:kv)
:ok
iex> Application.stop(:logger)
:ok
```

И давайте попробуем запустить наше приложение снова:

```iex
iex> Application.start(:kv)
{:error, {:not_started, :logger}}
```

Сейчас мы получаем ошибку, потому что какая-то из зависимостей `:kv` (в данном случае `:logger`) не запущена. Нам нужно также запустить каждое приложение внучную в правильном порядке или вызвать `Application.ensure_all_started` как показано ниже:

```iex
iex> Application.ensure_all_started(:kv)
{:ok, [:logger, :kv]}
```

Ничего удивительного не произошло, но мы увидели, как можем контроллировать наше приложение.

> Когда вы запускаете `iex -S mix`, это эквивалентно запуску `iex -S mix run`. Когда вам нужно передать больше опций в Mix при запуске IEx, важно написать именно `iex -S mix run` и затем передать любые опции, которые принимает команда `run`. Вы можете получить больше информации о `run` с помощью `mix help run` в вашей консоли.

## The application callback

Since we spent all this time talking about how applications are started and stopped, there must be a way to do something useful when the application starts. And indeed, there is!

We can specify an application callback function. This is a function that will be invoked when the application starts. The function must return a result of `{:ok, pid}`, where `pid` is the process identifier of a supervisor process.

We can configure the application callback in two steps. First, open up the `mix.exs` file and change `def application` to the following:

```elixir
  def application do
    [
      extra_applications: [:logger],
      mod: {KV, []}
    ]
  end
```

The `:mod` option specifies the "application callback module", followed by the arguments to be passed on application start. The application callback module can be any module that implements the [Application](https://hexdocs.pm/elixir/Application.html) behaviour.

Now that we have specified `KV` as the module callback, we need to change the `KV` module, defined in `lib/kv.ex`:

```elixir
defmodule KV do
  use Application

  def start(_type, _args) do
    KV.Supervisor.start_link(name: KV.Supervisor)
  end
end
```

When we `use Application`, we need to define a couple functions, similar to when we used `Supervisor` or `GenServer`. This time we only need to define a `start/2` function. If we wanted to specify custom behaviour on application stop, we could define a `stop/1` function.

Let's start our project console once again with `iex -S mix`. We will see a process named `KV.Registry` is already running:

```iex
iex> KV.Registry.create(KV.Registry, "shopping")
:ok
iex> KV.Registry.lookup(KV.Registry, "shopping")
{:ok, #PID<0.88.0>}
```

How do we know this is working? After all, we are creating the bucket and then looking it up; of course it should work, right? Well, remember that `KV.Registry.create/2` uses `GenServer.cast/2`, and therefore will return `:ok` regardless of whether the message finds its target or not. At that point, we don't know whether the supervisor and the server are up, and if the bucket was created. However, `KV.Registry.lookup/2` uses `GenServer.call/3`, and will block and wait for a response from the server. We do get a positive response, so we know all is up and running.

For an experiment, try reimplementing `KV.Registry.create/2` to use `GenServer.call/3` instead, and momentarily disable the application callback. Run the code above on the console again, and you will see the creation step fails straight away.

Don't forget to bring the code back to normal before resuming this tutorial!

## Projects or applications?

Mix makes a distinction between projects and applications. Based on the contents of our `mix.exs` file, we would say we have a Mix project that defines the `:kv` application. As we will see in later chapters, there are projects that don't define any application.

When we say "project" you should think about Mix. Mix is the tool that manages your project. It knows how to compile your project, test your project and more. It also knows how to compile and start the application relevant to your project.

When we talk about applications, we talk about <abbr title="Open Telecom Platform">OTP</abbr>. Applications are the entities that are started and stopped as a whole by the runtime. You can learn more about applications and how they relate to booting and shutting down of your system as a whole in the [docs for the Application module](https://hexdocs.pm/elixir/Application.html).

Next let's learn about one special type of supervisor that is designed to start and shut down children dynamically, called simple one for one.

