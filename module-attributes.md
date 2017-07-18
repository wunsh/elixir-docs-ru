---
title: Атрибуты модулей
---

# {{ page.title }}

Атрибуты модулей в Эликсире решают три задачи:

1. Они служат для аннотирования модуля, часто с информацией, котороя может использоваться пользователем или виртуальной машиной.
2. Они работают как константы.
3. Они работают как временное хранилище модуля, для использования во время компиляции.

Поговорим о каждом случае отдельно.

## Аннотации

Эликсир унаследовал концепцию аттрибутов модулей из Эрланга: Например:

```elixir
defmodule MyServer do
  @vsn 2
end
```

В примере выше мы добавляем атрибут с версией модуля. `@vsn` используется в механизме перезагрузки кода, в виртуальной машине Эрланга для проверки, был ли модуль обновлён. Если версия не указана, в её качестве устанавливается контрольная сумма MD5 функций модуля.

В эликсире есть пригоршня зарезервированных атрибутов. Вот некоторые из них, наиболее часто используемые:

* `@moduledoc` - для предоставления документации к текущему модулю.
* `@doc` - для документации к функции или макросу, следующим за атрибутом.
* `@behaviour` - (обратите внимание на Британское написание) используется для определения <abbr title="Open Telecom Platform">OTP</abbr> или определённого пользователем поведения.
* `@before_compile` - для хука, который будет выполнен до компиляции модуля. Благодаря этому возможно внедрять функции внутрь модуля прямо перед компиляцией.

`@moduledoc` и `@doc` используются чаще всего, и мы ожидаем, что вы тоже будете часто использовать их. Документация очень важна в Эликсире, есть множество функций для доступа к ней. Вы можете больше прочитать о [написании документации в Эликсире в официальной документации](https://hexdocs.pm/elixir/writing-documentation.html).

Давайте вернёмся к модулю `Math`, который мы определили в предыдущих главах, добавим к нему документацию и сохраним файл `math.ex`:

```elixir
defmodule Math do
  @moduledoc """
  Provides math-related functions.

  ## Examples

      iex> Math.sum(1, 2)
      3

  """

  @doc """
  Calculates the sum of two numbers.
  """
  def sum(a, b), do: a + b
end
```

Эликсир поощряет использование разметки Markdown и синтаксиса Heredoc для написания читаемой документации. Heredoc - многострочные строки, они начинаются и заканчиваются утроенными двойными кавычками, сохраняя форматирование внутреннего текста. Мы можем получить доступ к документации любого скомпилированного модуля прямо из IEx:

```bash
$ elixirc math.ex
$ iex
```

```iex
iex> h Math # Access the docs for the module Math
...
iex> h Math.sum # Access the docs for the sum function
...
```

Также существует инструмент [ExDoc](https://github.com/elixir-lang/ex_doc), который используется для генерации HTML страниц из документации.

Вы можете обратиться к документации по [модулям](https://hexdocs.pm/elixir/Module.html), чтобы получить полный список поддерживаемых атрибутов. Эликсир также использует атрибуты для определения [typespecs](/getting-started/typespecs-and-behaviours.html) (корректный перевод?).

Эта секция рассказывает о встроенных атрибутах Однако, атрибуты могут быть использованы разработчиками или расширены библиотеками для поддержки кастомного поведения.

## As constants

Elixir developers will often use module attributes as constants:

```elixir
defmodule MyServer do
  @initial_state %{host: "127.0.0.1", port: 3456}
  IO.inspect @initial_state
end
```

> Note: Unlike Erlang, user defined attributes are not stored in the module by default. The value exists only during compilation time. A developer can configure an attribute to behave closer to Erlang by calling [`Module.register_attribute/3`](https://hexdocs.pm/elixir/Module.html#register_attribute/3).

Trying to access an attribute that was not defined will print a warning:

```elixir
defmodule MyServer do
  @unknown
end
warning: undefined module attribute @unknown, please remove access to @unknown or explicitly set it before access
```

Finally, attributes can also be read inside functions:

```elixir
defmodule MyServer do
  @my_data 14
  def first_data, do: @my_data
  @my_data 13
  def second_data, do: @my_data
end

MyServer.first_data #=> 14
MyServer.second_data #=> 13
```

Every time an attribute is read inside a function, a snapshot of its current value is taken. In other words, the value is read at compilation time and not at runtime. As we are going to see, this also makes attributes useful to be used as storage during module compilation.

## As temporary storage

One of the projects in the Elixir organization is [the `Plug` project](https://github.com/elixir-lang/plug), which is meant to be a common foundation for building web libraries and frameworks in Elixir.

The Plug library also allows developers to define their own plugs which can be run in a web server:

```elixir
defmodule MyPlug do
  use Plug.Builder

  plug :set_header
  plug :send_ok

  def set_header(conn, _opts) do
    put_resp_header(conn, "x-header", "set")
  end

  def send_ok(conn, _opts) do
    send(conn, 200, "ok")
  end
end

IO.puts "Running MyPlug with Cowboy on http://localhost:4000"
Plug.Adapters.Cowboy.http MyPlug, []
```

In the example above, we have used the `plug/1` macro to connect functions that will be invoked when there is a web request. Internally, every time you call `plug/1`, the Plug library stores the given argument in a `@plugs` attribute. Just before the module is compiled, Plug runs a callback that defines a function (`call/2`) which handles HTTP requests. This function will run all plugs inside `@plugs` in order.

In order to understand the underlying code, we'd need macros, so we will revisit this pattern in the meta-programming guide. However the focus here is on how using module attributes as storage allows developers to create DSLs.

Another example comes from [the ExUnit framework](https://hexdocs.pm/ex_unit/) which uses module attributes as annotation and storage:

```elixir
defmodule MyTest do
  use ExUnit.Case

  @tag :external
  test "contacts external service" do
    # ...
  end
end
```

Tags in ExUnit are used to annotate tests. Tags can be later used to filter tests. For example, you can avoid running external tests on your machine because they are slow and dependent on other services, while they can still be enabled in your build system.

We hope this section shines some light on how Elixir supports meta-programming and how module attributes play an important role when doing so.

In the next chapters we'll explore structs and protocols before moving to exception handling and other constructs like sigils and comprehensions.