---
title: Спецификации типов и поведения
---

# {{ page.title }}

## Типы и спецификации

Эликсир - язык с динамической типизацией, поэтому все типы в Эликсире определяются во время выполнения. В эликсире есть **спецификации типов**, которые являются нотациями для:

1. определения сигнатур типизированных функции (спецификации);
2. определения пользовательских типов данных.

### Спецификации функций

По умолчанию в Эликсире есть несколько базовых типов, таких как `integer` или `pid`, а также более сложные типы: например, функция `round/1`, которая округляет число с плавающей точкой до ближайшего целого, принимает `number` в качестве аргумента (`integer` или `float`) и возвращает `integer`. Как вы можете увидеть в [ее документации](https://hexdocs.pm/elixir/Kernel.html#round/1), сигнатура `round/1` выглядит следующим образом:

```
round(number) :: integer
```

`::` означает, что функция слева возвращает значение того типа, который указан справа. Спецификации функций пишутся с помощью директивы `@spec`, расположенной прямо перед определением функции. Функция `round/1` может быть написана так:

```elixir
@spec round(number) :: integer
def round(number), do: # implementation...
```

Эликсир также поддерживает составные типы. Например, список целых чисел будет выглядеть как `[integer]`. Вы можете увидеть все встроенные типы Эликсира в [документации по спецификациям типов](https://hexdocs.pm/elixir/typespecs.html).

### Определение пользовательских типов

Эликсир предоставляет множество удобных встроенных типов, и также удобно в некоторых ситуациях определять пользовательские типы. Это можно сделать при определении модулей с помощью директивы `@type`.

Допустим, у нас есть модуль `LousyCalculator`, который умеет выполнять обычные арифметические операции (сложение, умножение и т.д.), но, вместо возврата чисел, он возвращает кортежи с результатом операции первым элементом и рандомно цитатой вторым.

```elixir
defmodule LousyCalculator do
  @spec add(number, number) :: {number, String.t}
  def add(x, y), do: {x + y, "You need a calculator to do that?!"}

  @spec multiply(number, number) :: {number, String.t}
  def multiply(x, y), do: {x * y, "Jeez, come on!"}
end
```

Как вы можете увидеть в примере, кортежи - составные типы, каждый кортеж идентифицируется типами внутри него. Чтобы понять, почему `String.t` не записано как `string`, взгляните на [заметки в документации по спецификациям типов](https://hexdocs.pm/elixir/typespecs.html#notes).

Определение спецификации функции таким способом работает, но быстро станет надоедать, если придётся повторять `{number, String.t}` снова и снова. Мы можем использовать директиву `@type` для объявления нашего пользовательского типа.

```elixir
defmodule LousyCalculator do
  @typedoc """
  Just a number followed by a string.
  """
  @type number_with_remark :: {number, String.t}

  @spec add(number, number) :: number_with_remark
  def add(x, y), do: {x + y, "You need a calculator to do that?"}

  @spec multiply(number, number) :: number_with_remark
  def multiply(x, y), do: {x * y, "It is like addition on steroids."}
end
```

Директива `@typedoc` используется по аналогии с `@doc` и `@moduledoc`, только для определения пользовательских типов.

Типы, определённые через `@type` экспортируются и становятся доступны снаружи модуля, в котором они определены:

```elixir
defmodule QuietCalculator do
  @spec add(number, number) :: number
  def add(x, y), do: make_quiet(LousyCalculator.add(x, y))

  @spec make_quiet(LousyCalculator.number_with_remark) :: number
  defp make_quiet({num, _remark}), do: num
end
```

Если вы хотите оставить пользовательский тип приватным, вы можете использовать директиву `@typep` вместо `@type`.

### Статистический анализ кода

Typespecs are not only useful to developers as additional documentation. The Erlang tool [Dialyzer](http://www.erlang.org/doc/man/dialyzer.html), for example, uses typespecs in order to perform static analysis of code. That's why, in the `QuietCalculator` example, we wrote a spec for the `make_quiet/1` function even though it was defined as a private function.

Спецификации типов полезны для разработчиков как дополнительная документация. Инструмент из Эрланга под названием [Dialyzer](http://www.erlang.org/doc/man/dialyzer.html), например, использует спецификации типов для статического анализа кода. Именно поэтому в примере `QuietCalculator`, мы писали спцификацию для функции `make_quiet/1`, хотя она была приватной.

## Поведения (behaviours)

Многие модули предоставляют одинаковый публичный API. Взгляните на [Plug](https://github.com/elixir-lang/plug) который, согласно его описанию, является **спецификацией** для компонуемых модулей в веб приложениях. Каждый *plug* - это модуль, который **должен** реализовывать как минимум две публичных функции: `init/1` и `call/2`.

Поведения предосатвляют нужны для того, чтобы:

* define a set of functions that have to be implemented by a module;
* ensure that a module implements all the functions in that set.

* определить набора функций, который модуль должен реализовывать;
* убедиться, что модуль реализует все функции из рабора.

Если нужно, вы можете думать о поведениях как о подобии интерфейсов из объектно-ориентированных языков, вроде Java: набор сигнатур функций, который модуль обязан реализовывать.

### Определение поведений

Допустим, мы хотим сделать несколько парсеров, каждый должен парсить структурированные данные: например, JSON и YAML. Каждый из этих двух парсеров будет вести себя подобным образом: оба будут реализовывать функции `parse/1` и `extensions/0`. Функция `parse/1` будет возвращать представление этих данных в Эликсире, тогда как `extensions/0` будет возвращать список файловых расширений, которые можно использовать для каждого типа данных (напирмер, `.json` для JSON файлов).

Мы можем создать поведение `Parser`:

```elixir
defmodule Parser do
  @callback parse(String.t) :: any
  @callback extensions() :: [String.t]
end
```

Модули, принимающие поведение `Parser` должны будут иметь все функции, заданные директивой `@callback`. Как вы видите, `@callback` принимает имя функции, а также её спецификацию, аналогично директиве `@spec`, рассмотренной выше.

### Adopting behaviours

Adopting a behaviour is straightforward:

```elixir
defmodule JSONParser do
  @behaviour Parser

  def parse(str), do: # ... parse JSON
  def extensions, do: ["json"]
end
```

```elixir
defmodule YAMLParser do
  @behaviour Parser

  def parse(str), do: # ... parse YAML
  def extensions, do: ["yml"]
end
```

If a module adopting a given behaviour doesn't implement one of the callbacks required by that behaviour, a compile-time warning will be generated.

