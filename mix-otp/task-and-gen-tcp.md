---
title: Task и gen_tcp
---

# {{ page.title }}

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

```iex
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

## Tasks

We have learned about agents, generic servers, and supervisors. They are all meant to work with multiple messages or manage state. But what do we use when we only need to execute some task and that is it?

[The Task module](https://hexdocs.pm/elixir/Task.html) provides this functionality exactly. For example, it has a `start_link/3` function that receives a module, function, and arguments, allowing us to run a given function as part of a supervision tree.

Let's give it a try. Open up `lib/kv_server/application.ex`, and let's change the supervisor in the `start/2` function to the following:

```elixir
  def start(_type, _args) do
    children = [
      {Task, fn -> KVServer.accept(4040) end}
    ]

    opts = [strategy: :one_for_one, name: KVServer.Supervisor]
    Supervisor.start_link(children, opts)
  end
```

With this change, we are saying that we want to run `KVServer.accept(4040)` as a task. We are hardcoding the port for now but this could be changed in a few ways, for example, by reading the port out of the system environment when starting the application:

```elixir
port = String.to_integer(System.get_env("PORT") || raise "missing $PORT environment variable")
# ...
{Task, fn -> KVServer.accept(port) end}
```

Now that the server is part of the supervision tree, it should start automatically when we run the application. Type `mix run --no-halt` in the terminal, and once again use the `telnet` client to make sure that everything still works:

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

Yes, it works! However, does it *scale*?

Try to connect two telnet clients at the same time. When you do so, you will notice that the second client doesn't echo:

```bash
$ telnet 127.0.0.1 4040
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
hello
hello?
HELLOOOOOO?
```

It doesn't seem to work at all. That's because we are serving requests in the same process that are accepting connections. When one client is connected, we can't accept another client.

## Task supervisor

In order to make our server handle simultaneous connections, we need to have one process working as an acceptor that spawns other processes to serve requests. One solution would be to change:

```elixir
defp loop_acceptor(socket) do
  {:ok, client} = :gen_tcp.accept(socket)
  serve(client)
  loop_acceptor(socket)
end
```

to use `Task.start_link/1`, which is similar to `Task.start_link/3`, but it receives an anonymous function instead of module, function and arguments:

```elixir
defp loop_acceptor(socket) do
  {:ok, client} = :gen_tcp.accept(socket)
  Task.start_link(fn -> serve(client) end)
  loop_acceptor(socket)
end
```

We are starting a linked Task directly from the acceptor process. But we've already made this mistake once. Do you remember?

This is similar to the mistake we made when we called `KV.Bucket.start_link/1` straight from the registry. That meant a failure in any bucket would bring the whole registry down.

The code above would have the same flaw: if we link the `serve(client)` task to the acceptor, a crash when serving a request would bring the acceptor, and consequently all other connections, down.

We fixed the issue for the registry by using a simple one for one supervisor. We are going to use the same tactic here, except that this pattern is so common with tasks that `Task` already comes with a solution: a simple one for one supervisor that starts temporary tasks as part of our supervision tree.

Let's change `start/2` once again, to add a supervisor to our tree:

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

We'll now start a [`Task.Supervisor`](https://hexdocs.pm/elixir/Task.Supervisor.html) process with name `KVServer.TaskSupervisor`. Remember, since the acceptor task depends on this supervisor, the supervisor must be started first.

Now we need to change `loop_acceptor/1` to use `Task.Supervisor` to serve each request:

```elixir
defp loop_acceptor(socket) do
  {:ok, client} = :gen_tcp.accept(socket)
  {:ok, pid} = Task.Supervisor.start_child(KVServer.TaskSupervisor, fn -> serve(client) end)
  :ok = :gen_tcp.controlling_process(client, pid)
  loop_acceptor(socket)
end
```

You might notice that we added a line, `:ok = :gen_tcp.controlling_process(client, pid)`. This makes the child process the "controlling process" of the `client` socket. If we didn't do this, the acceptor would bring down all the clients if it crashed because sockets would be tied to the process that accepted them (which is the default behaviour).

Start a new server with `PORT=4040 mix run --no-halt` and we can now open up many concurrent telnet clients. You will also notice that quitting a client does not bring the acceptor down. Excellent!

Here is the full echo server implementation:

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

Since we have changed the supervisor specification, we need to ask: is our supervision strategy still correct?

In this case, the answer is yes: if the acceptor crashes, there is no need to crash the existing connections. On the other hand, if the task supervisor crashes, there is no need to crash the acceptor too.

However, there is still one concern left, which are the restart strategies. Tasks, by default, have the `:restart` value set to `:temporary`, which means they are not restarted. This is an excellent default for the connections started via the `Task.Supervisor`, as it makes no sense to restart a failed connection, but it is a bad choice for the acceptor. If the acceptor crashes, we want to bring the acceptor up and running again.

We could fix this by defining our own module that calls `use Task, restart: :permanent` and invokes a `start_link` function responsible for restarting the task, quite similar to `Agent` and `GenServer`. However, let's take a different approach here. When integrating with someone else's library, we won't be able to change how their agents, tasks, and servers are defined. Instead, we need to be able to customize their child specification dynamically. This can be done by using `Supervisor.child_spec/2`, a function that we happen to know from previous chapters. Let's rewrite `start/2` in `KVServer.Application` once more:

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

`Supervisor.child_spec/2` is capable of building a child specification from a given module and/or tuple, and it also accepts values that override the underlying child specification. Now we have an always running acceptor that starts temporary task processes under an always running task supervisor.

In the next chapter, we will start parsing the client requests and sending responses, finishing our server.

