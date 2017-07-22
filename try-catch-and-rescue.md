---
title: try, catch и rescue
---

# {{ page.title }}

В Эликсире есть три механизма работы с непредвиденным поведением: ошибки, выбрасывание исключений и выходы. В этой главе мы рассмотрим каждый из них и случаи, когда использовать те или иные механизмы.

## Ошибки

Ошибки (или *исключения) используются, когда в коде происходят исключительные ситуации. Пример ошибки можно увидеть при попытке добавить число к атому:

```iex
iex> :foo + 1
** (ArithmeticError) bad argument in arithmetic expression
     :erlang.+(:foo, 1)
```

Ошибка рантайма может быть вызвана с помощью `raise/1`:

```iex
iex> raise "oops"
** (RuntimeError) oops
```

Другие ошибки можно вызвать, передав имя ошибки и список аргументов с ключами в `raise/2`:

```iex
iex> raise ArgumentError, message: "invalid argument foo"
** (ArgumentError) invalid argument foo
```

Вы также можете объявить собственные ошибки, создав модуль и использовав конструкцию `defexception` внутри него; таким образом вы создадите ошибку с тем же именем, что и у модуля, в котором она объявлена. Наиболее распространенный случай - объявление исключений с полем message:

```iex
iex> defmodule MyError do
iex>   defexception message: "default message"
iex> end
iex> raise MyError
** (MyError) default message
iex> raise MyError, message: "custom message"
** (MyError) custom message
```

Ошибки могут быть обработаны с помощью конструкции `try/rescue`:

```iex
iex> try do
...>   raise "oops"
...> rescue
...>   e in RuntimeError -> e
...> end
%RuntimeError{message: "oops"}
```

В примере выше ошибка рантайма отлавливается и возвращается для вывода в сессии `iex`.

Если у вас нет причины использовать саму ошибку, не обязательно её предоставлять:

```iex
iex> try do
...>   raise "oops"
...> rescue
...>   RuntimeError -> "Error!"
...> end
"Error!"
```

На практике, однако, Эликсир-разработчики редко используют конструкцию `try/rescue`. Например, многие языки обязали бы вас отловить ошибку, когда файл не может быть открыт. Эликсир же предоставляет функцию `File.read/1`, которая возвращает кортеж с информацие о том, что файл открыт успешно:

```iex
iex> File.read "hello"
{:error, :enoent}
iex> File.write "hello", "world"
:ok
iex> File.read "hello"
{:ok, "world"}
```

Здесь нет `try/rescue`. Если вы хотите обрабатывать различные исходы попытки открыть файл, можно использовать паттерн матчинг с конструкцией `case`:

```iex
iex> case File.read "hello" do
...>   {:ok, body}      -> IO.puts "Success: #{body}"
...>   {:error, reason} -> IO.puts "Error: #{reason}"
...> end
```

В конце концов, ваше приложение должно само решать, является ли ошибка при открытии файла исключительно или нет. Поэтому Элеисир не создаёт исключений в `File.read/1` и многих других функциях. Напротив, разработчик сам волен выбирать лучший способ поведения.

Для случаев, когда вы ожидаете, что файл существует (и отсутствие этого файла действительно *ошибка*), вы можете использовать `File.read!/1`:

```iex
iex> File.read! "unknown"
** (File.Error) could not read file unknown: no such file or directory
    (elixir) lib/file.ex:305: File.read!/1
```

Многие функции в стандартной библиотеке следуют схеме наличия двух вариантов функции, одна из которых вызывает исключения, вместо возврата кортежа. Договорённость состоит в том, чтобы была функция (`foo`), которая возввращает кортеж `{:ok, result}` или `{:error, reason}`, и другая функция (`foo!`, такое же имя с `!` в конце), которая принимает те же аргументы, что и `foo`, но вызывает исключения в случае ошибки. `foo!` должна возвращать результат (не обёрнтый кортежем), если всё прошло хорошо. [Модуль `File`](https://hexdocs.pm/elixir/File.html) - хороший пример следования этой договорённости.

В Эликсире мы избегаем использования `try/rescue`, потому что **не используем ошибки для контроля работы приложения**. Мы понимаем ошибки буквально: они оставлены для неожиданных и исключительных ситуаций. В случае, если вам действительно нужно контроллировать ход работы, следует использовать выбрасывание исключений с помощью *throw*. О нём мы и поговорим.

## Throws

В Эликсире, некоторое значение может быть выброшено (thrown) и далее отловлено (caught). `throw` и `catch` зарезервированы для ситуации, когда невозможно получить значение без использования `throw` и `catch`.

Такие ситуации на практике встречаются очень не часто, исключая случае работы с библиотеками, которые не предоставляют нормального API. Например, представьте, что модуль `Enum` не предоставлят API для поиска значений и что нам нужно найти первое кратное 13 число в списке чисел:

```iex
iex> try do
...>   Enum.each -50..50, fn(x) ->
...>     if rem(x, 13) == 0, do: throw(x)
...>   end
...>   "Got nothing"
...> catch
...>   x -> "Got #{x}"
...> end
"Got -39"
```

Но т.к. `Enum` *предоставляет* хороший API, на практике задача решается с использованием `Enum.find/2`:

```iex
iex> Enum.find -50..50, &(rem(&1, 13) == 0)
-39
```

## Exits

All Elixir code runs inside processes that communicate with each other. When a process dies of "natural causes" (e.g., unhandled exceptions), it sends an `exit` signal. A process can also die by explicitly sending an exit signal:

```iex
iex> spawn_link fn -> exit(1) end
#PID<0.56.0>
** (EXIT from #PID<0.56.0>) 1
```

In the example above, the linked process died by sending an `exit` signal with value of 1. The Elixir shell automatically handles those messages and prints them to the terminal.

`exit` can also be "caught" using `try/catch`:

```iex
iex> try do
...>   exit "I am exiting"
...> catch
...>   :exit, _ -> "not really"
...> end
"not really"
```

Using `try/catch` is already uncommon and using it to catch exits is even more rare.

`exit` signals are an important part of the fault tolerant system provided by the Erlang <abbr title="Virtual Machine">VM</abbr>. Processes usually run under supervision trees which are themselves processes that listen to `exit` signals from the supervised processes. Once an exit signal is received, the supervision strategy kicks in and the supervised process is restarted.

It is exactly this supervision system that makes constructs like `try/catch` and `try/rescue` so uncommon in Elixir. Instead of rescuing an error, we'd rather "fail fast" since the supervision tree will guarantee our application will go back to a known initial state after the error.

## After

Sometimes it's necessary to ensure that a resource is cleaned up after some action that could potentially raise an error. The `try/after` construct allows you to do that. For example, we can open a file and use an `after` clause to close it--even if something goes wrong:

```iex
iex> {:ok, file} = File.open "sample", [:utf8, :write]
iex> try do
...>   IO.write file, "olá"
...>   raise "oops, something went wrong"
...> after
...>   File.close(file)
...> end
** (RuntimeError) oops, something went wrong
```

The `after` clause will be executed regardless of whether or not the tried block succeeds. Note, however, that if a linked process exits,
this process will exit and the `after` clause will not get run. Thus `after` provides only a soft guarantee. Luckily, files in Elixir are also linked to the current processes and therefore they will always get closed if the current process crashes, independent of the
`after` clause. You will find the same to be true for other resources like ETS tables, sockets, ports and more.

Sometimes you may want to wrap the entire body of a function in a `try` construct, often to guarantee some code will be executed afterwards. In such cases, Elixir allows you to omit the `try` line:

```iex
iex> defmodule RunAfter do
...>   def without_even_trying do
...>     raise "oops"
...>   after
...>     IO.puts "cleaning up!"
...>   end
...> end
iex> RunAfter.without_even_trying
cleaning up!
** (RuntimeError) oops
```

Elixir will automatically wrap the function body in a `try` whenever one of `after`, `rescue` or `catch` is specified.

## Else

If an `else` block is present, it will match on the results of the `try` block whenever the `try` block finishes without a throw or an error.

```iex
iex> x = 2
2
iex> try do
...>   1 / x
...> rescue
...>   ArithmeticError ->
...>     :infinity
...> else
...>   y when y < 1 and y > -1 ->
...>     :small
...>   _ ->
...>     :large
...> end
:small
```

Exceptions in the `else` block are not caught. If no pattern inside the `else` block matches, an exception will be raised; this exception is not caught by the current `try/catch/rescue/after` block.

## Variables scope

It is important to bear in mind that variables defined inside `try/catch/rescue/after` blocks do not leak to the outer context. This is because the `try` block may fail and as such the variables may never be bound in the first place. In other words, this code is invalid:

```iex
iex> try do
...>   raise "fail"
...>   what_happened = :did_not_raise
...> rescue
...>   _ -> what_happened = :rescued
...> end
iex> what_happened
** (RuntimeError) undefined function: what_happened/0
```

Instead, you can store the value of the `try` expression:

```iex
iex> what_happened =
...>   try do
...>     raise "fail"
...>     :did_not_raise
...>   rescue
...>     _ -> :rescued
...>   end
iex> what_happened
:rescued
```

This finishes our introduction to `try`, `catch`, and `rescue`. You will find they are used less frequently in Elixir than in other languages, although they may be handy in some situations where a library or some particular code is not playing "by the rules".