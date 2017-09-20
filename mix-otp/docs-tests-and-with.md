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

```elixir
iex> KVServer.Command.parse "UNKNOWN shopping eggs\r\n"
{:error, :unknown_command}

iex> KVServer.Command.parse "GET shopping\r\n"
{:error, :unknown_command}
```

Без добавления пустых строк ExUnit скомпилирует их в один тест:

```elixir
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

## Запуск команд

Последний шаг - написать `KVServer.Command.run/1` для запуска команд после парсинга в приложении `:kv`. Реализация представлена ниже:

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

Каждый вариант функции отправляет нужную команду на сервер `KV.Registry`, который мы зарегистрировали при запуске приложения `:kv`. Т.к. наш `:kv_server` зависит от приложения `:kv`, он также зависит от предоставляемых последним сервисов.

Обратите внимание, что мы также определили приватную функцию `lookup/2`, чтобы переиспользовать общую функциональность по поиску корзины и возврату её `pid`, если она существует, или `{:error, :not_found}` в ином случае.

Кстати, т.к. теперь мы возвращаем `{:error, :not_found}`, следует изменить функцию `write_line/2` в `KVServer`, чтобы она выводила эту ошибку:

```elixir
defp write_line(socket, {:error, :not_found}) do
  :gen_tcp.send(socket, "NOT FOUND\r\n")
end
```

Функциональность нашего сервера почти закончена. Остались только тесты. Сейчас мы оставим тесты на самый конец, потому что есть несколько важных вещей, которым стоит уделить внимание.

`KVServer.Command.run/1` отправляет команды напрямую в сервер `KV.Registry`, который зарегистрирован приложением `:kv`. Это значит, что сервер доступен глобально и если у нас будет два теста, отправляющих сообщения в одно время, тесты будут конфликтовать друг с другом (и скорее всего упадут). Нам нужно решить, использовать юнит тесты, которые изолированы и могут быть запущены асинхронно, или писать интеграционные тесты, которые используют глобальное состояние, но испытывают наше приложение полностью, как оно будет работать в продакшне.

До сих пор мы писали только юнит тесты, обычно тестировали отдельно взятый модуль. Однако, чтобы сделать тестируемым `KVServer.Command.run/1` как юнит, нам придётся изменить его реализацию, не посылать команды в процесс `KV.Registry` напрямую, а передавать сервер как аргумент. Например, нам бы пришлось изменить сигнатуру `run` на `def run(command, pid)` и затем изменить все варианты функции:

```elixir
def run({:create, bucket}, pid) do
  KV.Registry.create(pid, bucket)
  {:ok, "OK\r\n"}
end

# ... other run clauses ...
```

Можете попробовать продолжить, сделать изменения, описанные выше, и написать юнит тесты. Идея в том, что наши тесты будут стартовать экземпляр `KV.Registry` и передавать его аргументом в `run/2` вместо обращений к глобальному `KV.Registry`. Это позволит сохранить тесты асинхронными, т.к. у них не будет общего состояния.

Но также давайте попробуем кое-что иное. Напишем интеграционные тесты, которые основаны на глобальном сервере, чтобы испытать весь стэк от TCP сервера и до корзин. Наши интеграционные тесты будут полагаться на глобальное состояние и должны быть синхронными. С интеграционными тестами мы будем знать, как компоненты нашего приложения работают друг с другом. Обычно так тестируют основные сценарии работы вашего приложения. Например, нет смысла писать интеграционные тесты для реализации парсинга.

Наш интеграционный тест будет использовать клиент TCP, который будет отправлять команды нашему серверу и утверждать, какой ответ мы хотим получить.

Давайте реализуем интеграционный тест `test/kv_server_test.exs` как показано ниже:

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

Наш интеграционный тест проверяет все взаимодействия с сервером, учитывая неизвестные команды и ошибки "not found". Мы используем таблицы <abbr title="Erlang Term Storage">ETS</abbr> и связанные процессы, поэтому даже нет необходимости закрывать сокет. Как только тестовый процесс закончится, сокет закроется автоматически.

Т.к. теперь наш тест основан на глобальном состоянии, мы не передаём `async: true` в `use ExUnit.Case`. Более того, чтобы гарантировать нашим тестам одинаковое чистое исходное состояние, мы останавливаем и запускаем приложение `:kv` перед каждым тестом. Остановка `:kv` пишет предупреждение в терминал:

```
18:12:10.698 [info] Application kv exited: :stopped
```

Чтобы избежать печати логов во время тестов, в ExUnit есть изящное решение `:capture_log`. Если установить `@tag :capture_log` перед каждым тестом или `@moduletag :capture_log` перед всем набором тестов, ExUnit будет автоматически отлавливать весь лог во время работы тестов. Если тест провалится, отловленный лог будет выведен в отчёте ExUnit.

Между `use ExUnit.Case` и запуском добавьте следующую строку:

```elixir
@moduletag :capture_log
```

Если тест упадёт, вы увидете отчёт вроде этого:

```
  1) test server interaction (KVServerTest)
     test/kv_server_test.exs:17
     ** (RuntimeError) oops
     stacktrace:
       test/kv_server_test.exs:29

     The following output was logged:

     13:44:10.035 [info]  Application kv exited: :stopped
```

С этим простым интеграционным тестом мы можем увидеть, почему интеграционные тесты могут быть медленными. Не только потому, что такой тест не может работать асинхронно, он также предусматривает дорогой установку запуска и старта приложения `:kv`.

В конце концов, ваша задача и задача вашей команды понять, какая стратегия тестирования будет лучшей для вашего приложения. Вам нужно найти баланс между качеством кода, уверенностью в его корректной работе и временем выполнения тестов. Например, мы можем выполнять только интеграционные тесты, но если сервер продолжает расти в следующих релизах, или становится часть приложения с частыми багами, важно иметь возможность разделить его на части и написать более быстрые и дешевые юнит тесты.

В следующей главе мы наконец сделаем нашу систему распределяемой, добавив механизм маршрутизации корзин. А также поговорим о конфигурации приложений.
