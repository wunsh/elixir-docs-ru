---
title: Доктесты, паттерны и `with`
---

# {{ page.title }}

В этой главе мы реализуем код, который парсит команды, описанные в первой главе:

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

Когда парсинг будет закончен, мы обновим наш сервер для отправки распарсенных команд в приложение `:kv`, созданное ранее.

## Доктесты (doctests)

В самом начале мы упомянули, что документация является гражданином первого класса в языке. Мы подходили к этому с разных сторон по ходу этого руководство, будь то `mix help`, или `h Enum`, или справка по другим модулям в консоли.

В этом разделе мы реализуем парсинг с использованием доктестов, которые позволяют нам писать тесты прямо в нашей документации. Это помогает нам предоставлять документацию сразу с примерами кода.

Давайте создадим модуль `lib/kv_server/command.ex` с парсером, и начнём с доктеста к нему:

```elixir
defmodule KVServer.Command do
  @doc ~S"""
  Parses the given `line` into a command.

  ## Examples

      iex> KVServer.Command.parse "CREATE shopping\r\n"
      {:ok, {:create, "shopping"}}

  """
  def parse(_line) do
    :not_implemented
  end
end
```

Доктесты задаются с помощью отступа в четыре пробела перед `iex>`. Если команда занимает несколько строк, можно использовать `...>`, также как IEx. Ожидаемый результат должен начинаться на следующей строке после строки `iex>` или `...>`, и заканчиваться новой строкой или новым префиксом `iex>`.

Также обратите внимание, что мы начале строку документации с помощью `@doc ~S"""`. `~S` предовращает конвертацию символов `\r\n` в возват каретки при разборе документации, пока они не будут достигнуты в тесте.

Для запуска нашего доктеста мы создадим файл `test/kv_server/command_test.exs` и вызовем `doctest KVServer.Command`:

```elixir
defmodule KVServer.CommandTest do
  use ExUnit.Case, async: true
  doctest KVServer.Command
end
```

Запустим тесты, доктест должен упасть:

```
  1) test doc at KVServer.Command.parse/1 (1) (KVServer.CommandTest)
     test/kv_server/command_test.exs:3
     Doctest failed
     code: KVServer.Command.parse "CREATE shopping\r\n" === {:ok, {:create, "shopping"}}
     lhs:  :not_implemented
     stacktrace:
       lib/kv_server/command.ex:7: KVServer.Command (module)
```

Прекрасно!

Теперь сделаем так, чтобы доктест проходил успешно. Реализуем функцию `parse/1`:

```elixir
def parse(line) do
  case String.split(line) do
    ["CREATE", bucket] -> {:ok, {:create, bucket}}
  end
end
```

Наша реализация разбивает линию по пробелам, затем ищет команду в списке. Использование `String.split/1` означает, что наши команды будут нечувствительны к пробелам. Пробелы в начале и конце будут проигнорированы, как и множественные пробелы между словами. Давайте добавим несколько новых доктестов, чтобы протестировать это поведение вместе с остальными командами:

```elixir
@doc ~S"""
Parses the given `line` into a command.

## Examples

    iex> KVServer.Command.parse "CREATE shopping\r\n"
    {:ok, {:create, "shopping"}}

    iex> KVServer.Command.parse "CREATE  shopping  \r\n"
    {:ok, {:create, "shopping"}}

    iex> KVServer.Command.parse "PUT shopping milk 1\r\n"
    {:ok, {:put, "shopping", "milk", "1"}}

    iex> KVServer.Command.parse "GET shopping milk\r\n"
    {:ok, {:get, "shopping", "milk"}}

    iex> KVServer.Command.parse "DELETE shopping eggs\r\n"
    {:ok, {:delete, "shopping", "eggs"}}

Unknown commands or commands with the wrong number of
arguments return an error:

    iex> KVServer.Command.parse "UNKNOWN shopping eggs\r\n"
    {:error, :unknown_command}

    iex> KVServer.Command.parse "GET shopping\r\n"
    {:error, :unknown_command}

"""
```

Когда доктесты написаны, ваша задача самостоятельно сделать так, чтобы они проходили! Когда закончите, можете сравнить вашу работу с нашим решением ниже:

```elixir
def parse(line) do
  case String.split(line) do
    ["CREATE", bucket] -> {:ok, {:create, bucket}}
    ["GET", bucket, key] -> {:ok, {:get, bucket, key}}
    ["PUT", bucket, key, value] -> {:ok, {:put, bucket, key, value}}
    ["DELETE", bucket, key] -> {:ok, {:delete, bucket, key}}
    _ -> {:error, :unknown_command}
  end
end
```

Посмотрите, как мы смогли элегантно парсить команды без добавления кучи `if/else`, которые проверяют имя команды и количество аргументов!

Наконец, вы можете увидеть, что каждый доктест воспринимается как отдельный тест, наш тестовый набор теперь сообщает о том, что всего их 7. Это происходит, потому что ExUnit воспринимает следующее как два разных теста:

```iex
iex> KVServer.Command.parse "UNKNOWN shopping eggs\r\n"
{:error, :unknown_command}

iex> KVServer.Command.parse "GET shopping\r\n"
{:error, :unknown_command}
```

Без добавления пустых строк ExUnit скомпилирует их в один тест:

```iex
iex> KVServer.Command.parse "UNKNOWN shopping eggs\r\n"
{:error, :unknown_command}
iex> KVServer.Command.parse "GET shopping\r\n"
{:error, :unknown_command}
```

Вы можете прочитать больше о доктестах в [документации `ExUnit.DocTest`](https://hexdocs.pm/ex_unit/ExUnit.DocTest.html).

## with

Так как теперь мы можем парсить команды, мы наконец приступаем к реализации логики, которая будет запускать их. Давайте пока добавим определение-заглушку для этой функции:

```elixir
defmodule KVServer.Command do
  @doc """
  Runs the given command.
  """
  def run(command) do
    {:ok, "OK\r\n"}
  end
end
```

До того, как мы реализуем эту функцию, давайте изменим наш сервер, чтобы он использовал наши новые функции `parse/1` и `run/1`. Вспомните, наша функция `read_line/1` также падала, когда клиент закрывал сокет, так что давайте попробуем заодно решить и эту проблему. Откройте `lib/kv_server.ex` и замените существующее определение сервера:

```elixir
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

на следующее:

```elixir
defp serve(socket) do
  msg =
    case read_line(socket) do
      {:ok, data} ->
        case KVServer.Command.parse(data) do
          {:ok, command} ->
            KVServer.Command.run(command)
          {:error, _} = err ->
            err
        end
      {:error, _} = err ->
        err
    end

  write_line(socket, msg)
  serve(socket)
end

defp read_line(socket) do
  :gen_tcp.recv(socket, 0)
end

defp write_line(socket, {:ok, text}) do
  :gen_tcp.send(socket, text)
end

defp write_line(socket, {:error, :unknown_command}) do
  # Known error. Write to the client.
  :gen_tcp.send(socket, "UNKNOWN COMMAND\r\n")
end

defp write_line(_socket, {:error, :closed}) do
  # The connection was closed, exit politely.
  exit(:shutdown)
end

defp write_line(socket, {:error, error}) do
  # Unknown error. Write to the client and exit.
  :gen_tcp.send(socket, "ERROR\r\n")
  exit(error)
end
```

Если мы запустим наш сервер, мы можем теперь отправлять в него команды. Пока мы будем получать два разных ответа: "OK" когда команда известна и "UNKNOWN COMMAND" в ином случае:

```bash
$ telnet 127.0.0.1 4040
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
CREATE shopping
OK
HELLO
UNKNOWN COMMAND
```

Это значит, что наш вариант идёт в верном направлении, но это не выглядит достаточно элегантно, не так ли?

Предыдущая реализация использовала пайплайны, которые делали логику прямолинейной. Однако, теперь нам нужно обрабатывать разные коды ошибок, а логика нашего сервера вложена во много вызовов `case`.

К счастью, Эликсир v1.2 представил конструкцию `with`, которая позволяет вам упростить код вроде примера выше, заменив вложенные вызовы `case` на цепочку условий сравнения. Давайте перепишем функцию `serve/1`, используя `with`:

```elixir
defp serve(socket) do
  msg =
    with {:ok, data} <- read_line(socket),
         {:ok, command} <- KVServer.Command.parse(data),
         do: KVServer.Command.run(command)

  write_line(socket, msg)
  serve(socket)
end
```

Намного лучше! `with` получит значение, возвращённое правой стороной `<-` и сравнит его с шаблоном слева. Если значение подходит под шаблон, `with` перейдёт к следующему выражению. Если совпадения нет, результат, не прошедший сравнение, будет возвращён.

Другими словами, мы преобразовали каждое выражение, переданное в `case/2` в шаг внутри `with`. Как только любой из шагов возвращает что-то отличное от `{:ok, x}`, `with` прерывается и возвращает это не прошедшее сравнение значение.

Вы можете прочитать больше о [`with` в нашей документации](https://hexdocs.pm/elixir/Kernel.SpecialForms.html#with/1).

## Running commands

The last step is to implement `KVServer.Command.run/1`, to run the parsed commands against the `:kv` application. Its implementation is shown below:

```elixir
@doc """
Runs the given command.
"""
def run(command)

def run({:create, bucket}) do
  KV.Registry.create(KV.Registry, bucket)
  {:ok, "OK\r\n"}
end

def run({:get, bucket, key}) do
  lookup bucket, fn pid ->
    value = KV.Bucket.get(pid, key)
    {:ok, "#{value}\r\nOK\r\n"}
  end
end

def run({:put, bucket, key, value}) do
  lookup bucket, fn pid ->
    KV.Bucket.put(pid, key, value)
    {:ok, "OK\r\n"}
  end
end

def run({:delete, bucket, key}) do
  lookup bucket, fn pid ->
    KV.Bucket.delete(pid, key)
    {:ok, "OK\r\n"}
  end
end

defp lookup(bucket, callback) do
  case KV.Registry.lookup(KV.Registry, bucket) do
    {:ok, pid} -> callback.(pid)
    :error -> {:error, :not_found}
  end
end
```

Every function clause dispatches the appropriate command to the `KV.Registry` server that we registered during the `:kv` application startup. Since our `:kv_server` depends on the `:kv` application, it is completely fine to depend on the services it provides.

Note that we have also defined a private function named `lookup/2` to help with the common functionality of looking up a bucket and returning its `pid` if it exists, `{:error, :not_found}` otherwise.

By the way, since we are now returning `{:error, :not_found}`, we should amend the `write_line/2` function in `KVServer` to print such error as well:

```elixir
defp write_line(socket, {:error, :not_found}) do
  :gen_tcp.send(socket, "NOT FOUND\r\n")
end
```

Our server functionality is almost complete. Only tests are missing. This time, we have left tests for last because there are some important considerations to be made.

`KVServer.Command.run/1`'s implementation is sending commands directly to the server named `KV.Registry`, which is registered by the `:kv` application. This means this server is global and if we have two tests sending messages to it at the same time, our tests will conflict with each other (and likely fail). We need to decide between having unit tests that are isolated and can run asynchronously, or writing integration tests that work on top of the global state, but exercise our application's full stack as it is meant to be exercised in production.

So far we have only written unit tests, typically testing a single module directly. However, in order to make `KVServer.Command.run/1` testable as a unit we would need to change its implementation to not send commands directly to the `KV.Registry` process but instead pass a server as argument. For example, we would need to change `run`'s signature to `def run(command, pid)` and then change all clauses accordingly:

```elixir
def run({:create, bucket}, pid) do
  KV.Registry.create(pid, bucket)
  {:ok, "OK\r\n"}
end

# ... other run clauses ...
```

Feel free to go ahead and do the changes above and write some unit tests. The idea is that your tests will start an instance of the `KV.Registry` and pass it as argument to `run/2` instead of relying on the global `KV.Registry`. This has the advantage of keeping our tests asynchronous as there is no shared state.

But let's also try something different. Let's write integration tests that rely on the global server names to exercise the whole stack from the TCP server to the bucket. Our integration tests will rely on global state and must be synchronous. With integration tests we get coverage on how the components in our application work together at the cost of test performance. They are typically used to test the main flows in your application. For example, we should avoid using integration tests to test an edge case in our command parsing implementation.

Our integration test will use a TCP client that sends commands to our server and assert we are getting the desired responses.

Let's implement the integration test in `test/kv_server_test.exs` as shown below:

```elixir
defmodule KVServerTest do
  use ExUnit.Case

  setup do
    Application.stop(:kv)
    :ok = Application.start(:kv)
  end

  setup do
    opts = [:binary, packet: :line, active: false]
    {:ok, socket} = :gen_tcp.connect('localhost', 4040, opts)
    %{socket: socket}
  end

  test "server interaction", %{socket: socket} do
    assert send_and_recv(socket, "UNKNOWN shopping\r\n") ==
           "UNKNOWN COMMAND\r\n"

    assert send_and_recv(socket, "GET shopping eggs\r\n") ==
           "NOT FOUND\r\n"

    assert send_and_recv(socket, "CREATE shopping\r\n") ==
           "OK\r\n"

    assert send_and_recv(socket, "PUT shopping eggs 3\r\n") ==
           "OK\r\n"

    # GET returns two lines
    assert send_and_recv(socket, "GET shopping eggs\r\n") == "3\r\n"
    assert send_and_recv(socket, "") == "OK\r\n"

    assert send_and_recv(socket, "DELETE shopping eggs\r\n") ==
           "OK\r\n"

    # GET returns two lines
    assert send_and_recv(socket, "GET shopping eggs\r\n") == "\r\n"
    assert send_and_recv(socket, "") == "OK\r\n"
  end

  defp send_and_recv(socket, command) do
    :ok = :gen_tcp.send(socket, command)
    {:ok, data} = :gen_tcp.recv(socket, 0, 1000)
    data
  end
end
```

Our integration test checks all server interaction, including unknown commands and not found errors. It is worth noting that, as with <abbr title="Erlang Term Storage">ETS</abbr> tables and linked processes, there is no need to close the socket. Once the test process exits, the socket is automatically closed.

This time, since our test relies on global data, we have not given `async: true` to `use ExUnit.Case`. Furthermore, in order to guarantee our test is always in a clean state, we stop and start the `:kv` application before each test. In fact, stopping the `:kv` application even prints a warning on the terminal:

```
18:12:10.698 [info] Application kv exited: :stopped
```

To avoid printing log messages during tests, ExUnit provides a neat feature called `:capture_log`. By setting `@tag :capture_log` before each test or `@moduletag :capture_log` for the whole test case, ExUnit will automatically capture anything that is logged while the test runs. In case our test fails, the captured logs will be printed alongside the ExUnit report.

Between `use ExUnit.Case` and setup, add the following call:

```elixir
@moduletag :capture_log
```

In case the test crashes, you will see a report as follows:

```
  1) test server interaction (KVServerTest)
     test/kv_server_test.exs:17
     ** (RuntimeError) oops
     stacktrace:
       test/kv_server_test.exs:29

     The following output was logged:

     13:44:10.035 [info]  Application kv exited: :stopped
```

With this simple integration test, we start to see why integration tests may be slow. Not only this test cannot run asynchronously, it also requires the expensive setup of stopping and starting the `:kv` application.

At the end of the day, it is up to you and your team to figure out the best testing strategy for your applications. You need to balance code quality, confidence, and test suite runtime. For example, we may start with testing the server only with integration tests, but if the server continues to grow in future releases, or it becomes a part of the application with frequent bugs, it is important to consider breaking it apart and writing more intensive unit tests that don't have the weight of an integration test.

In the next chapter we will finally make our system distributed by adding a bucket routing mechanism. We'll also learn about application configuration.
