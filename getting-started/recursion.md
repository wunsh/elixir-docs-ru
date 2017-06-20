# Рекурсия

## Циклы через рекурсию

Ввиду иммутабельности, циклы в Elixir (как и в других функциональных языках программирования) отличаются в написании от императивных языков. Например, в C-подобных языках они могут быть написаны так:

```c
for(i = 0; i < sizeof(array); i++) {
  array[i] = array[i] * 2;
}
```

В примере выше мы изменяем и массив, и переменную `i`. Изменение невозможно в Elixir. Функциональные языки в противовес полагаются на рекурсию: функция вызывается рекурсивно, пока условие для остановки не будет достигнуто. Никакие данные в процессе не изменяются. Посмотрите на пример ниже, который печатает строку указанное количество раз:

```elixir
defmodule Recursion do
  def print_multiple_times(msg, n) when n <= 1 do
    IO.puts msg
  end

  def print_multiple_times(msg, n) do
    IO.puts msg
    print_multiple_times(msg, n - 1)
  end
end

Recursion.print_multiple_times("Hello!", 3)
# Hello!
# Hello!
# Hello!
```

Аналогично `case`, функция может иметь несколько вариантов вызова. Определённый вариант будет выполнен, когда переданные аргументы удовлетворяют шаблону аргументов и его ограничивающие условия возвращают `true`.

При вызове `print_multiple_times/2` в примере выше, аргумент `n` равен `3`.

Первый вариант имеет ограничение, которое говорит "используй это определение тогда и только тогда, когда `n` меньше или равно `1`". Т.к. в начале этот вариант не подходит, используется следующее определение.

Второе определение подходит шаблону и не имеет ограничений, таким образом оно будет выполнено. При этом будет напечатано наше `msg` и будет вызвана эта же функция с `n - 1` (`2`) в качестве второго аргумента.

Наше `msg` напечатано и `print_multiple_times/2` вызывается снова, на этот раз со вторым аргументом `1`. Т.к. `n` теперь равно `1`, ограничение из первого определения вернёт true и выполнится соответствующий ему код. `msg` будет напечатано и больше ничего не останется для выполнения.

Мы определили `print_multiple_times/2` так, что неважно, какое число будет передано в качестве второго аргумента, будет или выполнено первое определение (его можно назвать базовым случаем) или второе, которое приблизит нас на шаг к базовому случаю.

## Reduce and map algorithms

Let's now see how we can use the power of recursion to sum a list of numbers:

```elixir
defmodule Math do
  def sum_list([head | tail], accumulator) do
    sum_list(tail, head + accumulator)
  end

  def sum_list([], accumulator) do
    accumulator
  end
end

IO.puts Math.sum_list([1, 2, 3], 0) #=> 6
```

We invoke `sum_list` with the list `[1, 2, 3]` and the initial value `0` as arguments. We will try each clause until we find one that matches according to the pattern matching rules. In this case, the list `[1, 2, 3]` matches against `[head | tail]` which binds `head` to `1` and `tail` to `[2, 3]`; `accumulator` is set to `0`.

Then, we add the head of the list to the accumulator `head + accumulator` and call `sum_list` again, recursively, passing the tail of the list as its first argument. The tail will once again match `[head | tail]` until the list is empty, as seen below:

```elixir
sum_list [1, 2, 3], 0
sum_list [2, 3], 1
sum_list [3], 3
sum_list [], 6
```

When the list is empty, it will match the final clause which returns the final result of `6`.

The process of taking a list and _reducing_ it down to one value is known as a _reduce algorithm_ and is central to functional programming.

What if we instead want to double all of the values in our list?

```elixir
defmodule Math do
  def double_each([head | tail]) do
    [head * 2 | double_each(tail)]
  end

  def double_each([]) do
    []
  end
end
```

```bash
iex math.exs
```

```iex
iex> Math.double_each([1, 2, 3]) #=> [2, 4, 6]
```

Here we have used recursion to traverse a list, doubling each element and returning a new list. The process of taking a list and _mapping_ over it is known as a _map algorithm_.

Recursion and [tail call](https://en.wikipedia.org/wiki/Tail_call) optimization are an important part of Elixir and are commonly used to create loops. However, when programming in Elixir you will rarely use recursion as above to manipulate lists.

The [`Enum` module](https://hexdocs.pm/elixir/Enum.html), which we're going to see in the next chapter, already provides many conveniences for working with lists. For instance, the examples above could be written as:

```iex
iex> Enum.reduce([1, 2, 3], 0, fn(x, acc) -> x + acc end)
6
iex> Enum.map([1, 2, 3], fn(x) -> x * 2 end)
[2, 4, 6]
```

Or, using the capture syntax:

```iex
iex> Enum.reduce([1, 2, 3], 0, &+/2)
6
iex> Enum.map([1, 2, 3], &(&1 * 2))
[2, 4, 6]
```

Let's take a deeper look at `Enumerable`s and, while we're at it, their lazy counterpart, `Stream`s.