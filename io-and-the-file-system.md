---
title: Ввод/вывод и файловая система
---

# {{ page.title }}

Эта глава - быстрое введение в механизмы ввода/вывода и задачи, связанные с файловой системой, а также модулями [`IO`](https://hexdocs.pm/elixir/IO.html), [`File`](https://hexdocs.pm/elixir/File.html) и [`Path`](https://hexdocs.pm/elixir/Path.html).

Изначально мы планировали разместить эту главу намного ближе к началу руководства. Однако, мы поняли, что система ввода/вывода даёт прекрасную возможность пролить свет на некоторые философские и любопытные вещи в Elixir и виртуальной машине.

## Модуль `IO`

Модуль [`IO`](http://elixir-lang.org/docs/v1.0/elixir/IO.html) - основной механизм Эликсира для работы со стандартным вводов/выводом (`:stdio`), стандартным выводом ошибок (`:stderr`), файлами, и другими устройствами IO. Его использование простое и очевидное:

```iex
iex> IO.puts "hello world"
hello world
:ok
iex> IO.gets "yes or no? "
yes or no? yes
"yes\n"
```

По умолчанию, функции из модуля `IO` читают стандартный ввод и пишут в стандартный вывод. Мы можем изменить это, передав, например, `:stderr` в качестве аргумента (чтобы осуществить запись в стандартное устройство вывода ошибок):

```iex
iex> IO.puts :stderr, "hello world"
hello world
:ok
```

## Модуль `File`

Модуль [`File`](https://hexdocs.pm/elixir/File.html) содержит функции, которые позволяют открывать файлы как IO устройства. По умолчанию, файлы открываются в бинарном режиме, что обязывает разработчиков использовать специальные функции `IO.binread/2` и `IO.binwrite/2` из модуля `IO`:

```iex
iex> {:ok, file} = File.open "hello", [:write]
{:ok, #PID<0.47.0>}
iex> IO.binwrite file, "world"
:ok
iex> File.close file
:ok
iex> File.read "hello"
{:ok, "world"}
```

Файл может быть также открыт с указанием кодировки `:utf8`, в этом случае модуль `File` будет интерпретировать байты, прочитанные из файла, как байты кодировки UTF-8.

Кроме функций для открытия, чтения и записи файлов, модуль `File` имеет много функций для работы с файловой системой. Эти функции названы соответственно их UNIX эквивалентам. Например, `File.rm/1` можно использовать для удаления файла, `File.mkdir/1` для создания директорий, `File.mkdir_p/1` для создания директорий и последовательности её предков. Есть также `File.cp_r/2` и `File.rm_rf/1` для копирования директорий со всем содержимым и рекурсивного удаления директории и всех её файлов.

Вы также можете обнаружить, что функции в модуле `File` представлены в двух вариантах: "обычный" вариант и вариант, принудительный вариант, оканчивающийся восклицательным знаком(`!`). Например, когда мы читаем файл `"hello"` в примере выше, мы используем `File.read/1`. Мы можем также использовать `File.read!/1`:

```iex
iex> File.read "hello"
{:ok, "world"}
iex> File.read! "hello"
"world"
iex> File.read "unknown"
{:error, :enoent}
iex> File.read! "unknown"
** (File.Error) could not read file "unknown": no such file or directory
```

Обратите внимание, что версия с `!` возвращает содержимое файла, а не кортеж, и если что-то идёт не так, выбрасывает ошибку.

Версия без `!` предпочтительна, если вы хотите обработать разные варианты вывода, используя сравнение с шаблоном:

```elixir
case File.read(file) do
  {:ok, body}      -> # do something with the `body`
  {:error, reason} -> # handle the error caused by `reason`
end
```

Однако, если вы уверены, что файл существует, принудительный вариант несёт больше пользы, т.к. выдаёт понятное сообщение об ошибке. Не пишите так:

```elixir
{:ok, body} = File.read(file)
```

т.к. в случае ошибки `File.read/1` вернёт `{:error, reason}`, и это не будет соответствовать шаблону, сравнение не пройдёт. Вы также получите желаемый результат (выброшенную ошибку), но она будет связана с отсутствием подходящего шаблона (что не даст нам понять, в чём же на самом деле проблема с файлом).

Поэтому, если вы не хотите обрабатывать ошибки, предпочтительнее использовать `File.read!/1`.

## Модуль `Path`

Большинство функций в модуле `File` принимают пути в качестве аргументов. Наиболее часто, эти пути будут regular binaries (?). Модуль [`Path`](https://hexdocs.pm/elixir/Path.html) предоставляет возможности для работы с такими путями:

```iex
iex> Path.join("foo", "bar")
"foo/bar"
iex> Path.expand("~/hello")
"/Users/jose/hello"
```

Использование функций из модуля `Path` вместо прямых манипуляций со строками является предпочтительным, т.к. модуль `Path` заботится о различиях операционных систем. Наконец, помните, что Эликсир автоматически конвертирует слэши (`/`) в обратные слэши (`\`) в Windows при исполнении файловых операций.

Теперь мы имеем представление об основных модулях, которые Эликсир предоставляет для работы со вводом/выводом и взаимодействия с файловой системой. В следующих секциях мы поговорим о более продвинутых темах, связанных с IO. Эти секции не являются обязательными для написания кода на Эликсире, поэтому их можно пропустить, но они дают представление, как реализована система ввода/вывода в виртуальной машине, и объясняют другие любопытные вещи.

## Процессы и лидеры групп

Вы могли заметить, что `File.open/2` возвращает кортеж вроде `{:ok, pid}`:

```iex
iex> {:ok, file} = File.open "hello", [:write]
{:ok, #PID<0.47.0>}
```

Это происходит, потому что модуль `IO` на самом деле работает с процессами(смотрите [главу 11](/getting-started/processes.html)). Когда вы пишете `IO.write(pid, binary)`, модуль `IO` отправляет сообщение процессу с идентификатором `pid` с желаемой операцией. Давайте посмотрим, что происходит, если мы используем наш собственный процесс:

```iex
iex> pid = spawn fn ->
...>  receive do: (msg -> IO.inspect msg)
...> end
#PID<0.57.0>
iex> IO.write(pid, "hello")
{:io_request, #PID<0.41.0>, #Reference<0.0.8.91>,
 {:put_chars, :unicode, "hello"}}
** (ErlangError) erlang error: :terminated
```

После `IO.write/2` мы можем увидеть запрос, отправленный модулем `IO` (кортеж из четырех элементов). Сразу после мы видим ошибку, т.к. модуль `IO` ожидает некоторый результат, который мы не предоставляем.

Модуль [`StringIO`](https://hexdocs.pm/elixir/StringIO.html) - реализация сообщений для устройств ввода-вывода поверх строк:

```iex
iex> {:ok, pid} = StringIO.open("hello")
{:ok, #PID<0.43.0>}
iex> IO.read(pid, 2)
"he"
```

При моделировании устройств ввода-вывода с процессами, виртуальная машина Эрланга позволяет разным узлам одной сети обмениваться файловыми процессами и читать/записывать файлы между узлами. Среди всех устройств ввода-вывода есть одно особое для каждого процесса: **лидер группы**.

Когда вы пишете в `:stdio`, вы на самом деле отправляете сообщение лидеру группы, который пишет в файловый дескриптор для стандартного вывода:

```iex
iex> IO.puts :stdio, "hello"
hello
:ok
iex> IO.puts Process.group_leader, "hello"
hello
:ok
```

Лидер группы можно задать для процесса и использовать в разных ситуациях. Например, при исполнении кода на удалённом терминале, он гарантирует, что сообщение на удалённом узле будет переправлено и напечатано на терминале, который обрабатывает запрос.

## `iodata` and `chardata`

In all of the examples above, we used binaries when writing to files. In the chapter ["Binaries, strings and char lists"](/getting-started/binaries-strings-and-char-lists.html), we mentioned how strings are made of bytes while char lists are lists with unicode codepoints.

The functions in `IO` and `File` also allow lists to be given as arguments. Not only that, they also allow a mixed list of lists, integers and binaries to be given:

```iex
iex> IO.puts 'hello world'
hello world
:ok
iex> IO.puts ['hello', ?\s, "world"]
hello world
:ok
```

However, using list in IO operations requires some attention. A list may represent either a bunch of bytes or a bunch of characters and which one to use depends on the encoding of the IO device. If the file is opened without encoding, the file is expected to be in raw mode, and the functions in the `IO` module starting with `bin*` must be used. Those functions expect an `iodata` as argument; i.e., they expect a list of integers representing bytes and binaries to be given.

On the other hand, `:stdio` and files opened with `:utf8` encoding work with the remaining functions in the `IO` module. Those functions expect a `char_data` as an argument, that is, a list of characters or strings.

Although this is a subtle difference, you only need to worry about these details if you intend to pass lists to those functions. Binaries are already represented by the underlying bytes and as such their representation is always "raw".

This finishes our tour of IO devices and IO related functionality. We have learned about four Elixir modules - [`IO`](https://hexdocs.pm/elixir/IO.html), [`File`](https://hexdocs.pm/elixir/File.html), [`Path`](https://hexdocs.pm/elixir/Path.html) and [`StringIO`](https://hexdocs.pm/elixir/StringIO.html) - as well as how the <abbr title="Virtual Machine">VM</abbr> uses processes for the underlying IO mechanisms and how to use `chardata` and `iodata` for IO operations.