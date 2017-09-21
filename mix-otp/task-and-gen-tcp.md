---
title: Task и gen_tcp
next_page: mix-otp/docs-tests-and-with
prev_page: mix-otp/dependencies-and-umbrella-apps
---

В этой главе мы изучим, как использовать [Модуль Эрланга `:gen_tcp`](http://www.erlang.org/doc/man/gen_tcp.html) для обработки запросов. При этом мы получим возможность посмотреть также модуль Эликсира `Task`. В дальнейших главах мы расширим наш сервер так, чтобы он действительно мог обрабатывать команды.

## Эхо-сервер

Мы начнём наш TCP сервер с реализации эхо-сервера. Он будет посылать в ответ тот текст, который он получил в запросе. Мы будем постепенно улучшать наш сервер, пока он не будет управляться супервизором и будет готов к работе с множеством подключений.

TCP сервер, грубо говоря, делает следующие шаги:

1. Слушает порт, пока порт доступен и удерживает сокет
2. Ждёт подключение клиента на этом порту и принимает его
3. Читает запросы клиента и отправляет ответы

Давайте реализуем эти шаги. Перейдите к приложению `apps/kv_server`, откройте `lib/kv_server.ix` и добавьте следующие функции:

```elixir
require Logger

def accept(port) do
  # The options below mean:
  #
  # 1. `:binary` - receives data as binaries (instead of lists)
  # 2. `packet: :line` - receives data line by line
  # 3. `active: false` - blocks on `:gen_tcp.recv/2` until data is available
  # 4. `reuseaddr: true` - allows us to reuse the address if the listener crashes
  #
  {:ok, socket} = :gen_tcp.listen(port,
                    [:binary, packet: :line, active: false, reuseaddr: true])
  Logger.info "Accepting connections on port #{port}"
  loop_acceptor(socket)
end

defp loop_acceptor(socket) do
  {:ok, client} = :gen_tcp.accept(socket)
  serve(client)
  loop_acceptor(socket)
end

defp serve(socket) do
  socket
  |> read_line()
  |> write_line(socket)

  serve(socket)
end

defp read_line(socket) do
  {:ok, data} = :gen_tcp.recv(socket, 0)
  data
end

defp write_line(line, socket) do
  :gen_tcp.send(socket, line)
end
```

Мы запустим наш сервер, вызвав `KVServer.accept(4040)`, где 4040 - это порт. Первый шаг в  `accept/1` - слушать порт пока сокет не станет доступен и затем вызвать `loop_acceptor/1`. `loop_acceptor/1` - цикл, принимающий подключения клиентов. Для каждого принятого подключения мы вызываем `serve/1`.

`serve/1` - другой цикл, который читает строки из сокета и пишет эти строки обратно в сокет. Обратите внимание, что функция `serve/1` использует [оператор конвейера `|>`](https://hexdocs.pm/elixir/Kernel.html#%7C%3E/2) для выполнения этого потока операций. Оператор конвейера выполняет левую часть и передаёт её результат в качетве первого аргумента в функцию в правой части. Пример выше:

```elixir
socket |> read_line() |> write_line(socket)
```

эквивалентен этому:

```elixir
write_line(read_line(socket), socket)
```

Метод `read_line/1` получает данные из сокета, используя `:gen_tcp.recv/2`, и `write_line/2` пишет в сокет, используя `:gen_tcp.send/2`.

обратите внимание, что `serve/1` - это бесконечный цикл, вызываемый последовательно внутри `loop_acceptor/1`, поэтому конечный вызов `loop_acceptor/1` никогда не будет достигнут и его можно опустить. Однако, как мы увидим, нам нужно будет выполнять `serve/1` в отдельном процессе, поэтому нам скоро понадобится этот конечный вызов.

Это всё, чно нам нужно для реализации нашего эхо-сервера. Давайте попробуем его в деле!

Запустите сессию IEx внутри приложения `kv_server`, используя `iex -S mix`. В IEx запустите:

```elixir
iex> KVServer.accept(4040)
```

Сервер теперь запущен, и вы можете увидеть, что консоль заблокирована. Давайте используем [клиент `telnet`](https://en.wikipedia.org/wiki/Telnet) для доступа к нашему серверу. Такие клиенты доступны для многих операционных систем, а их команды обычно похожи:

```bash
$ telnet 127.0.0.1 4040
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
hello
hello
is it me
is it me
you are looking for?
you are looking for?
```

Введите "hello", нажмите Enter, и вы получите "hello" в ответ. Прекрасно!

Мой telnet клиент можно закрыть, нажав `ctrl + ]`, набрав следом `quit`, и нажав `<Enter>`, но ваш клиент может предусматривать другую последовательность действий.

Когда вы закроете клиент telnet, вы скорее всего увидите ошибку в сессии IEx:

    ** (MatchError) no match of right hand side value: {:error, :closed}
        (kv_server) lib/kv_server.ex:45: KVServer.read_line/1
        (kv_server) lib/kv_server.ex:37: KVServer.serve/1
        (kv_server) lib/kv_server.ex:30: KVServer.loop_acceptor/1

Это происходит, потому что мы ждём данные от `:gen_tcp.recv/2`, но клиент закрывает соединение. Нам нужно лучше обрабатывать подобные сценарии в будущих версиях нашего сервера.

Пока у нас есть более важные проблемы, которые нужно решить: что произойдёт, если наш приёмник TCP соединений упадёт? Т.к. у нас нет супервизора, сервер умрёт и мы не сможем больше обрабатывать запросы, потому что он не перезапустится. Поэтому нам необходимо поместить наш сервер в дерево супервизора.

## Задачи

Мы изучили агенты, генсерверы и супервизоры. Они все работают с множеством сообщений или управляют состоянием. Но что делать, если нам нужно всего лишь выполнить какую-то одну задачу?

[Модуль Task](https://hexdocs.pm/elixir/Task.html) предоставляет именно такую функциональность. Например, там есть фукнция `start_link/3`, которая принимает модуль, функцию и аргументы, позволяя нам запустить переданную фукнцию как часть дерева супервизора.

Давайте попробуем на практике. Откройте `lib/kv_server/application.ex` и измените супервизор в функции `start/2` как показано ниже:

```elixir
  def start(_type, _args) do
    children = [
      {Task, fn -> KVServer.accept(4040) end}
    ]

    opts = [strategy: :one_for_one, name: KVServer.Supervisor]
    Supervisor.start_link(children, opts)
  end
```

Этим изменением мы говорим, что хотим запустить `KVServer.accept(4040)` как задачу. Мы задали порт прямо в коде, но он может быть изменён несколькими способами, например, его можно взять из системного окружения при запуске приложения:

```elixir
port = String.to_integer(System.get_env("PORT") || raise "missing $PORT environment variable")
# ...
{Task, fn -> KVServer.accept(port) end}
```

Теперь сервер является частью деева супервизора и должен запуститься автоматически при запуске приложения. Наберите `mix run --no-halt` в консоли и снова используйте клиент `telnet`, чтобы убедиться, что всё работает:

```bash
$ telnet 127.0.0.1 4040
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
say you
say you
say me
say me
```

Да, всё отлично! Однако, может ли это решение *масштабироваться*?

Попробуйте подключить два клиента telnet одновременно. Когда вы сделаете это, вы увидите, что второй клиент не отвечает:

```bash
$ telnet 127.0.0.1 4040
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
hello
hello?
HELLOOOOOO?
```

Не похоже, что он вообще работает. это происходит потому, что мы обрабатываем запросы в том же процессе, в котором принимаем подключения. Когда один клиент подключен, мы не можем подключить другой клиент.

## Супервизор задач

Чтобы наш сервер мог обрабатывать множество подключений, нам нужно сделать так, чтобы один процесс принимал подключения и порождал другие процессы, которые обрабатывают запросы. Одно из решений - изменить код:

```elixir
defp loop_acceptor(socket) do
  {:ok, client} = :gen_tcp.accept(socket)
  serve(client)
  loop_acceptor(socket)
end
```

используя `Task.start_link/1`, которая похожа на `Task.start_link/3`, но принимает анонимную функцию вместо модуля, функции и аргументов:

```elixir
defp loop_acceptor(socket) do
  {:ok, client} = :gen_tcp.accept(socket)
  Task.start_link(fn -> serve(client) end)
  loop_acceptor(socket)
end
```

Мы запускаем связанную задачу прямо из процесса-приёмника. Но мы уже делали такую ошибку раньше. Вы помните?

Это такая же ошибка, как в случае с вызовом `KV.Bucket.start_link/1` прямо из реестра. Там падение любой корзины приводило к падению всего реестра.

Код выше имеет такую же проблему: если мы связываем задачу `serve(client)` с приёмником, падение обработки запроса приведёт к падению приёмника и всех других подключений.

Для реестра мы решили эту проблему, используя супервизор. Мы придержимся этой же тактики здесь, т.к. этот шаблон настолько распространён для задач, что модуль `Task` уже содержит решение: простой супервизор "one for one", который запускает временные задачи как часть нашего дерева супервизора.

Давайте изменим `start/2` ещё раз, добавив супервизор в наше дерево:

```elixir
  def start(_type, _args) do
    children = [
      {Task.Supervisor, name: KVServer.TaskSupervisor},
      {Task, fn -> KVServer.accept(4040) end}
    ]

    opts = [strategy: :one_for_one, name: KVServer.Supervisor]
    Supervisor.start_link(children, opts)
  end
```

Сейчас мы запустим процесс [`Task.Supervisor`](https://hexdocs.pm/elixir/Task.Supervisor.html) с именем `KVServer.TaskSupervisor`. Помните, т.к. задача приёмника зависит от нашего супервизора, супервизор должен быть запущен первым.

Теперь нам нужно изменить `loop_acceptor/1` с использованием `Task.Supervisor` для обработки каждого запроса:

```elixir
defp loop_acceptor(socket) do
  {:ok, client} = :gen_tcp.accept(socket)
  {:ok, pid} = Task.Supervisor.start_child(KVServer.TaskSupervisor, fn -> serve(client) end)
  :ok = :gen_tcp.controlling_process(client, pid)
  loop_acceptor(socket)
end
```

Вы можете заметить, что мы добавили строку `:ok = :gen_tcp.controlling_process(client, pid)`. Это сделает процесс-потомок "контроллирующим процессом" для сокета `client`. Если бы мы не сделали это, приёмник при падении отключил бы всех клиентов, потому что сокеты были бы связаны с процессом, который принимает их (это стандартное поведение).

Запустите новый сервер с помощью `PORT=4040 mix run --no-halt`, а следом откройте несколько telnet клиентов параллельно. Вы сможете убедиться, что отключение клиента не приводит к отключению приёмника. Прекрасно!

Вот полная реализация эхо-сервера:

```elixir
defmodule KVServer do
  require Logger

  @doc """
  Starts accepting connections on the given `port`.
  """
  def accept(port) do
    {:ok, socket} = :gen_tcp.listen(port,
                      [:binary, packet: :line, active: false, reuseaddr: true])
    Logger.info "Accepting connections on port #{port}"
    loop_acceptor(socket)
  end

  defp loop_acceptor(socket) do
    {:ok, client} = :gen_tcp.accept(socket)
    {:ok, pid} = Task.Supervisor.start_child(KVServer.TaskSupervisor, fn -> serve(client) end)
    :ok = :gen_tcp.controlling_process(client, pid)
    loop_acceptor(socket)
  end

  defp serve(socket) do
    socket
    |> read_line()
    |> write_line(socket)

    serve(socket)
  end

  defp read_line(socket) do
    {:ok, data} = :gen_tcp.recv(socket, 0)
    data
  end

  defp write_line(line, socket) do
    :gen_tcp.send(socket, line)
  end
end
```

Мы изменили спецификацию супервизора, теперь нужно спросить: является ли наша стратегия супервизора всё ещё правильной?

В данном случае ответ "да": если приёмник падает, нет необходимости разрушать все существующие подключения. С другой стороны, если задача супервизора падает, также нет нужды отключать приёмник.

Однако, осталась ещё одна проблема: стратегия перезапуска. Задачи, по умолчанию, имеют в поле `:restart` значение `:temporary`, то есть они не будут перезапущены. Это отличный вариант для соединений, запущенных через `Task.Supervisor`, т.к. нет смысла перезапускать упавшее подключение, но это плохой выбор для приёмника. Если он упадёт, мы хотим снова его запустить.

Мы можем исправить это, определив в нашем модуле вызов `use Task, restart: :permanent` и назначить функцию `start_link` ответственной за перезапуск, по аналогии с `Agent` и `GenServer`. Однако, давайте поступим другим образом в данном случае. При интеграции с чужой библиотекой, мы не сможем изменить там определение агентов, задач и серверов. В этом случае нам нужно иметь возможность изменить спецификацию их потомков динамически. Это можно сделать с помощью `Supervisor.child_spec/2`, функции, с которой мы познакомились в предыдущих главах. Давайте перепишем `start/2` в `KVServer.Application` ещё раз:

```elixir
  def start(_type, _args) do
    children = [
      {Task.Supervisor, name: KVServer.TaskSupervisor},
      Supervisor.child_spec({Task, fn -> KVServer.accept(4040) end}, restart: :permanent)
    ]

    opts = [strategy: :one_for_one, name: KVServer.Supervisor]
    Supervisor.start_link(children, opts)
  end
```

`Supervisor.child_spec/2` может собирать спецификации потомков из переданного модуля и/или кортежа, и также принимает значения, которые переопределяют существующую спецификацию потомков. Теперь у нас есть всегда запущенный приёмник, который запускает временные процессы задач через всегда запущенный супервизор задач.

В следующей главе мы сделаем парсер для запросов клиентов и отправку ответов на них, и доделаем наш сервер до конца.
