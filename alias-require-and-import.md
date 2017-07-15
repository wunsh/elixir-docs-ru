---
title: alias, require и import
---

# {{ page.title }}

Для обеспечения повторного использования ПО, Эликсир предоставляет три директивы (`alias`, `require` и `import`), а также макрос `use`, описанные ниже:

```elixir
# Псевдоним для модуля, можно обращаться к Bar вместо Foo.Bar
alias Foo.Bar, as: Bar

# Подключение модуля для использования его макросов
require Foo

# Импорт функций из Foo, чтобы вызывать их без префикса `Foo.`
import Foo

# Вызывает код из Foo в качестве точки расширения
use Foo
```

Мы рассмотрим их подробно. Помните, что первые три называются директивами, потому что имеют **лексический скоуп**, в то время как `use` - это точка расширения.

## alias

`alias` позволяет нам устанавливать псевдонимы для любых имён модулей.

Представьте модуль, который использует особую форму списка из `Math.List`. Директива `alias` позволяет ссылаться на `Math.List` по имени `List`, без указания модуля:

```elixir
defmodule Stats do
  alias Math.List, as: List
  # In the remaining module definition List expands to Math.List.
end
```

Оригинальный `List` также доступен внутри `Stats` по полному имени `Elixir.List`.

> Обратите внимание: Все модули, определённые в Эликсире, определены в главном пространстве имён `Elixir`. Однако, для удобства, вы можете опустить "Elixir." при обращении к ним.

Псевдонимы часто используются для сокращений. Вызов `alias` без опции `:as` автоматически устанавливает псевдоним для последней части имени модуля, например:

```elixir
alias Math.List
```

То же самое, что:

```elixir
alias Math.List, as: List
```

Помните, что `alias` имеет **лексическую область видимости**, поэтому вы можете устанавливать псевдонимы внутри функций:

```elixir
defmodule Math do
  def plus(a, b) do
    alias Math.List
    # ...
  end

  def minus(a, b) do
    # ...
  end
end
```

В примере выше, с момента, когда мы вызвали `alias` внутри функции `plus/2`, псевдоним будет работать внутри функции `plus/2`. При этом он не будет доступен внутри `minus/2`.

## require

Эликсир предоставляет макросы, как механизм для мета-программирования (написания кода, который генерирует код). Макросы генерируют код во время компиляции.

Публичные функции в модулях доступны глобально, но для использования макросов, вам нужно отказаться от использования модуля, в котором они определены.

```iex
iex> Integer.is_odd(3)
** (UndefinedFunctionError) function Integer.is_odd/1 is undefined or private. However there is a macro with the same name and arity. Be sure to require Integer if you intend to invoke this macro
iex> require Integer
Integer
iex> Integer.is_odd(3)
true
```

В Эликсире, `Integer.is_odd/1` объявлена как макрос, поэтому её можно использовать как ограничивающее условие. Это значит, что для её вызова нужно подключить (с помощью `require`) модуль `Integer`.

Обратите внимание, что как и `alias`, `require` имеет лексическую область видимости. Мы поговорим подробнее о макросах в следющей главе.

## import

Мы используем `import`, когда хотим получить лёгкий доступ к функциям или макросам из другого модуля, без использования полного имени. Например, если мы хотим использовать фукнцию `duplicate/2` из модуля `List` несколько раз, мы можем импортировать его:

```iex
iex> import List, only: [duplicate: 2]
List
iex> duplicate :ok, 3
[:ok, :ok, :ok]
```

В данном случае, мы импортируем только функцию `duplicate` (с арностью 2) из `List`. Хотя `:only` опционален, его использование рекомендовано во избежание импорта всех функций из модуля в текущее пространство имён. `:except` может также быть передан для импорта всех функций, кроме указанных.

`import` также поддерживает `:macros` и `:functions` для передачи в `:only`. Например, для импорта всех макросов можно написать:

```elixir
import Integer, only: :macros
```

Или, для импорта всех функций:

```elixir
import Integer, only: :functions
```

Помните, что `import` также имеет **лексическую область видимости**. Это значит, что мы можем импортировать указанные макросы и функции внутри определения функций:

```elixir
defmodule Math do
  def some_function do
    import List, only: [duplicate: 2]
    duplicate(:ok, 10)
  end
end
```

В примере выше, импортированная `List.duplicate/2` видна только внутри функции. `duplicate/2` не будет доступна в других функциях этого модуля (или любого другого).

Обратите внимание, что `import` модуля автоматически производит его `require`.

## use

Макрос `use` часто используется разработчиками для добавления внешней функциональности в текущую лексическую область видимости, зачастую, модуль.

Например, для написания тестов с использованием фрэймворка ExUnit, разраобтчику нужен модуль `ExUnit.Case`:

```elixir
defmodule AssertionTest do
  use ExUnit.Case, async: true

  test "always pass" do
    assert true
  end
end
```

По сути, `use` делает `require` и вызывает коллбэк `__using__/1` для него, позволяя модулю внедрить некоторый код в текущий контекст. В целом, следующий модуль:

```elixir
defmodule Example do
  use Feature, option: :value
end
```

компилируется в 

```elixir
defmodule Example do
  require Feature
  Feature.__using__(option: :value)
end
```

## Понимание псевдонимов

К этому моменту, вы, должно быть, задаётесь вопросом: чем являются псевдонимы в Эликсире и как они представлены?

Псевдоним в Эликсире - идентификатор, начинающийся с большой буквы (например, `String`, `Keyword` и т.д.), который конвертируется в атом во время компиляции. Например, псевдоним `String` переводится по умолчанию в атом `:"Elixir.String"`:

```iex
iex> is_atom(String)
true
iex> to_string(String)
"Elixir.String"
iex> :"Elixir.String" == String
true
```

Используя директиву `alias/2`, мы изменяем атом, в который разворачивается псевдоним.

Псевдонимы разворачиваются в атомы, потому что модули виртуальной машины Эрланга (и, соответственно, Эликсира) всегда представлены как атомы. Например, вот механизм вызова модулей Эрланга:

```iex
iex> :lists.flatten([1, [2], 3])
[1, 2, 3]
```

## Module nesting

Now that we have talked about aliases, we can talk about nesting and how it works in Elixir. Consider the following example:

```elixir
defmodule Foo do
  defmodule Bar do
  end
end
```

The example above will define two modules: `Foo` and `Foo.Bar`. The second can be accessed as `Bar` inside `Foo` as long as they are in the same lexical scope. The code above is exactly the same as:

```elixir
defmodule Elixir.Foo do
  defmodule Elixir.Foo.Bar do
  end
  alias Elixir.Foo.Bar, as: Bar
end
```

If, later, the `Bar` module is moved outside the `Foo` module definition, it must be referenced by its full name (`Foo.Bar`) or an alias must be set using the `alias` directive discussed above.

**Note**: in Elixir, you don't have to define the `Foo` module before being able to define the `Foo.Bar` module, as the language translates all module names to atoms. You can define arbitrarily-nested modules without defining any module in the chain (e.g., `Foo.Bar.Baz` without defining `Foo` or `Foo.Bar` first).

As we will see in later chapters, aliases also play a crucial role in macros, to guarantee they are hygienic.

## Multi alias/import/require/use

From Elixir v1.2, it is possible to alias, import or require multiple modules at once. This is particularly useful once we start nesting modules, which is very common when building Elixir applications. For example, imagine you have an application where all modules are nested under `MyApp`, you can alias the modules `MyApp.Foo`, `MyApp.Bar` and `MyApp.Baz` at once as follows:

```elixir
alias MyApp.{Foo, Bar, Baz}
```

With this we have finished our tour of Elixir modules. The last topic to cover is module attributes.