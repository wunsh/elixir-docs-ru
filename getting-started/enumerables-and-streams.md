---
layout: getting-started
title: Enumerables and Streams
---

# Перечисления и потоки

## Перечисления (enumerables)

Elixir позволяет работать с перечислениями с помощью [модуля `Enum`](https://hexdocs.pm/elixir/Enum.html). Мы уже познакомились с двумя перечисляемыми типами: списки и мэпы. 

```iex
iex> Enum.map([1, 2, 3], fn x -> x * 2 end)
[2, 4, 6]
iex> Enum.map(%{1 => 2, 3 => 4}, fn {k, v} -> k * v end)
[2, 12]
```

Модуль `Enum` предоставляет огромное множество функция для трансформации, сортировки, группировки, фильтрации и извлечения элементов из перечислений. Это один из наболее часто используемых в коде Elixir разработчиков модулей:

В Elixir также есть диапазоны:

```iex
iex> Enum.map(1..3, fn x -> x * 2 end)
[2, 4, 6]
iex> Enum.reduce(1..3, 0, &+/2)
6
```

Функции в модуле Enum ограничены, как и говорит название, перечислением значений в структурах данных. Для специфических операций, вроде добавления или обновления элементов, вам, возможно, понадобится модуль соответствующей структуры данных. Например, если вы хотите добавить элемент на необходимую позицию в списке, вам нужна функция `List.insert_at/3` из [модуля `List`](https://hexdocs.pm/elixir/List.html), потому что было бы мало смысла во вставке значения, например, в диапазон.

Можно сказать, что функции в модуле `Enum` полиморфны, потому что они могут работать с разными типами данных. В частности, функции из модуля `Enum` могут работать с любым типаом данных, который реализует [протокол `Enumerable`](https://hexdocs.pm/elixir/Enumerable.html). Мы поговорим о протоколах позднее; сейчас мы перейдём к особенному виду перечислений, называемых потоками.

## Жадная или ленивая работа

Все функции из модуля `Enum` жадные. Многие функции, ожидающие на вход перечисление, возвращают списки:

```iex
iex> odd? = &(rem(&1, 2) != 0)
#Function<6.80484245/1 in :erl_eval.expr/5>
iex> Enum.filter(1..3, odd?)
[1, 3]
```

Это значит, что при выполнении нескольких операций с `Enum`, каждая из них создаст промежуточный список, пока будет достигнут результат:

```iex
iex> 1..100_000 |> Enum.map(&(&1 * 3)) |> Enum.filter(odd?) |> Enum.sum
7500000000
```

В пример выше есть последовательность (pipeline) операций. Мы начинаем с диапазона, затем умножаем каждый его элемента на 3. Первая операция создаст и вернёт список со `100_000` элементов. Затем мы выбираем все нечётные элементы из этого списка, создаём новый список, теперь с `50_000` элементов, и затем суммируем их все.

## Оператор конвейера

Символ `|>` в коде выше - это **оператор конвейера** (pipe operator): он принимает вывод из выражения слева и передаёт его первым аргументом в вызов функции справа. Он аналогичен оператору `|` в Unix. Его задача - явно выделить данные, которые будут преобразованы несколькими функциями. Чтобы увидеть, как это сделает код чище, взгляните на пример выше, переписанный без оператора `|>`:

```iex
iex> Enum.sum(Enum.filter(Enum.map(1..100_000, &(&1 * 3)), odd?))
7500000000
```

Больше информации об операторе конвейера [в его документации](https://hexdocs.pm/elixir/Kernel.html#%7C%3E/2).

## Streams

As an alternative to `Enum`, Elixir provides [the `Stream` module](https://hexdocs.pm/elixir/Stream.html) which supports lazy operations:

```iex
iex> 1..100_000 |> Stream.map(&(&1 * 3)) |> Stream.filter(odd?) |> Enum.sum
7500000000
```

Streams are lazy, composable enumerables.

In the example above, `1..100_000 |> Stream.map(&(&1 * 3))` returns a data type, an actual stream, that represents the `map` computation over the range `1..100_000`:

```iex
iex> 1..100_000 |> Stream.map(&(&1 * 3))
#Stream<[enum: 1..100000, funs: [#Function<34.16982430/1 in Stream.map/2>]]>
```

Furthermore, they are composable because we can pipe many stream operations:

```iex
iex> 1..100_000 |> Stream.map(&(&1 * 3)) |> Stream.filter(odd?)
#Stream<[enum: 1..100000, funs: [...]]>
```

Instead of generating intermediate lists, streams build a series of computations that are invoked only when we pass the underlying stream to the `Enum` module. Streams are useful when working with large, *possibly infinite*, collections.

Many functions in the `Stream` module accept any enumerable as an argument and return a stream as a result. It also provides functions for creating streams. For example, `Stream.cycle/1` can be used to create a stream that cycles a given enumerable infinitely. Be careful to not call a function like `Enum.map/2` on such streams, as they would cycle forever:

```iex
iex> stream = Stream.cycle([1, 2, 3])
#Function<15.16982430/2 in Stream.cycle/1>
iex> Enum.take(stream, 10)
[1, 2, 3, 1, 2, 3, 1, 2, 3, 1]
```

On the other hand, `Stream.unfold/2` can be used to generate values from a given initial value:

```iex
iex> stream = Stream.unfold("hełło", &String.next_codepoint/1)
#Function<39.75994740/2 in Stream.unfold/2>
iex> Enum.take(stream, 3)
["h", "e", "ł"]
```

Another interesting function is `Stream.resource/3` which can be used to wrap around resources, guaranteeing they are opened right before enumeration and closed afterwards, even in the case of failures. For example, we can use it to stream a file:

```iex
iex> stream = File.stream!("path/to/file")
#Function<18.16982430/2 in Stream.resource/3>
iex> Enum.take(stream, 10)
```

The example above will fetch the first 10 lines of the file you have selected. This means streams can be very useful for handling large files or even slow resources like network resources.

The amount of functionality in the [`Enum`](https://hexdocs.pm/elixir/Enum.html) and [`Stream`](https://hexdocs.pm/elixir/Stream.html) modules can be daunting at first, but you will get familiar with them case by case. In particular, focus on the `Enum` module first and only move to `Stream` for the particular scenarios where laziness is required, to either deal with slow resources or large, possibly infinite, collections.

Next we'll look at a feature central to Elixir, Processes, which allows us to write concurrent, parallel and distributed programs in an easy and understandable way.