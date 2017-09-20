---
title: Предметно-ориентированные языки
---

## Введение

[Предметно-ориентированные языки (DSL)](https://ru.wikipedia.org/wiki/Предметно-ориентированный_язык) позволяют разработчикам адаптировать свои приложения для какой-то конкретной сферы. Вам не нужны макросы, чтобы пользоваться DSL: каждая структура данных и каждая функция, которую вы определяете в своём модуле, является частью вашего предметно-ориентированного языка.

Например, представьте себе, что мы хотим внедрить модуль Validator, который обеспечивает проверку достоверности данных в вашем языке. Мы могли бы реализовать его с использованием структур данных, функций или макросов. Давайте же посмотрим, как выглядят эти DSL:

```elixir
# 1. data structures
import Validator
validate user, name: [length: 1..100],
               email: [matches: ~r/@/]

# 2. functions
import Validator
user
|> validate_length(:name, 1..100)
|> validate_matches(:email, ~r/@/)

# 3. macros + modules
defmodule MyValidator do
  use Validator
  validate_length :name, 1..100
  validate_matches :email, ~r/@/
end

MyValidator.validate(user)
```

Из всех вышеперечисленных подходов, первый, безусловно, является наиболее гибким. Если нормы нашего предмета могут быть закодированы структурами данных, значит, их легче всего создавать и реализовывать, поскольку стандартная библиотека Эликсира просто напичкана функциями для управления различными типами данных.

Второй подход использует вызовы функций, которые лучше подходят для более сложных API, например, если вам необходимо передать множество опций. К тому же, он хорошо читается в Эликсире, благодаря оператору конвейера.

Третий подход использует макросы и, безусловно, является сложным. Для его реализации нам потребуется больше строк кода, что делает его более трудным для восприятия и дорогостоящим в тестировании, если сравнивать с тестированием простых функций. А ещё, он ограничивает пользователя при работе с библиотекой, поскольку все проверки обязаны определяться внутри модуля.

Чтобы лучше понять это, представьте, что вы хотите проверить определённый атрибут только лишь в том случае, если выполнено заданное условие. Мы могли бы легко сделать это с помощью первого решения, манипулируя соответственным образом структурой данных или же вообще, благодаря второму решению, используя условные операторы (if/else) перед вызовом функции. Однако, всё это невозможно сделать, используя подход с макросами, если его DSL не будет дополнен.

Другими словами:

```
data > functions > macros
```

Тем не менее, все ешё есть случаи, когда использование макросов и модулей для создания предметно-ориентированных языков оказывается чем-то полезным. Поскольку мы изучили структуры данных и определения функций в руководстве Getting Started, в этой главе будет рассмотрено, как использовать макросы и атрибуты модуля для решения проблем более сложных DSL.

# Создание нашего собственного тестового примера

Целью этой главы является создание модуля с именем `TestCase`, что позволяет нам написать следующее:

```elixir
defmodule MyTest do
  use TestCase

  test "arithmetic operations" do
    4 = 2 + 2
  end

  test "list operations" do
    [1, 2, 3] = [1, 2] ++ [3]
  end
end

MyTest.run
```

В приведённом выше примере, используя `TestCase`, мы можем писать тесты с помощью макроса `test`, который определяет функцию с именем `run` для автоматического запуска всех наших тестов. Таким образом, наш прототип будет полагаться на оператор соответствия (`=`) как механизм для выполнения заданных утверждений.

# Макрос `test`

Начнём с создания модуля, который определяет и импортирует макрос `test` при его использовании:

```elixir
defmodule TestCase do
  # Callback invoked by `use`.
  #
  # For now it returns a quoted expression that
  # imports the module itself into the user code.
  @doc false
  defmacro __using__(_opts) do
    quote do
      import TestCase
    end
  end

  @doc """
  Defines a test case with the given description.

  ## Examples

      test "arithmetic operations" do
        4 = 2 + 2
      end

  """
  defmacro test(description, do: block) do
    function_name = String.to_atom("test " <> description)
    quote do
      def unquote(function_name)(), do: unquote(block)
    end
  end
end
```

Предполагая, что мы определили `TestCase` в файле с именем `tests.exs`, мы можем открыть его, запустив `iex tests.exs` и, таким образом, определим наши первые тесты:

```elixir
iex> defmodule MyTest do
...>   use TestCase
...>
...>   test "hello" do
...>     "hello" = "world"
...>   end
...> end
```

На данный момент у нас нет механизма для запуска тестов и мы с вами знаем, что функция “test hello” была определена тайно. Поэтому, когда мы вызываем на ней наш тест, он должен не пройти:

```elixir
iex> MyTest."test hello"()
** (MatchError) no match of right hand side value: "world"
```

# Хранение информации с атрибутами

Чтобы завершить реализацию нашего `TestCase`, мы должны иметь доступ ко всем нашим тестам. Один из способов сделать это - получать тесты во время их выполнения с помощью `__MODULE__.__info__(:functions)`, который возвращает список всех функций в текущем модуле. Однако, учитывая, что мы можем хранить гораздо больше информации о каждом тесте, за исключением его имени, нам необходим более гибкий подход.

При обсуждении атрибутов модуля в предыдущих главах, мы упоминали, как их можно использовать в качестве временного хранилища. Именно это их свойство мы применим в этом разделе.

В реализации `__using__/1` мы инициализируем атрибут модуля с именем `@tests` в пустой список, а затем сохраняем имя каждого определённого теста на этом атрибуте, чтобы они могли быть вызваны из функции `run`.

Вот обновленный код модуля `Test Case`:

```elixir
defmodule TestCase do
  @doc false
  defmacro __using__(_opts) do
    quote do
      import TestCase

      # Initialize @tests to an empty list
      @tests []

      # Invoke TestCase.__before_compile__/1 before the module is compiled
      @before_compile TestCase
    end
  end

  @doc """
  Defines a test case with the given description.

  ## Examples

      test "arithmetic operations" do
        4 = 2 + 2
      end

  """
  defmacro test(description, do: block) do
    function_name = String.to_atom("test " <> description)
    quote do
      # Prepend the newly defined test to the list of tests
      @tests [unquote(function_name) | @tests]
      def unquote(function_name)(), do: unquote(block)
    end
  end

  # This will be invoked right before the target module is compiled
  # giving us the perfect opportunity to inject the `run/0` function
  @doc false
  defmacro __before_compile__(_env) do
    quote do
      def run do
        Enum.each @tests, fn name ->
          IO.puts "Running #{name}"
          apply(__MODULE__, name, [])
        end
      end
    end
  end
end
```

Начав новую сессию в IEx, мы теперь можем определить и запустить наши тесты:

```elixir
iex> defmodule MyTest do
...>   use TestCase
...>
...>   test "hello" do
...>     "hello" = "world"
...>   end
...> end
iex> MyTest.run
Running test hello
** (MatchError) no match of right hand side value: "world"
```

Несмотря на то, что мы не затронули некоторые детали, это основная идея создания предметно-ориентированных модулей в Эликсире. Макросы позволяют нам возвращать маскирующие выражения, выполняющиеся в вызывающем их выражении, которые мы затем можем использовать для преобразования кода и хранения соответствующей информации в целевом модуле с помощью его атрибутов. И наконец, обратные вызовы, такие как `@before_compile`, позволяют нам вводить код в модуль по завершению его определения.

Помимо `@before_compile`, существуют в модулях и другие полезные атрибуты, например, такие как `@on_definition` и `@after_compile`, о которых вы можете почитать в [документации для модуля `Module`](https://hexdocs.pm/elixir/Module.html). А также, вы можете найти полезную информацию о макросах и среде компиляции в документации для [модуля `Macro`](https://hexdocs.pm/elixir/Macro.html) и [`Macro.Env`](https://hexdocs.pm/elixir/Macro.Env.html).
