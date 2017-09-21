---
title: Протоколы
next_page: comprehensions
prev_page: structs
---

# {{ page.title }}

Протоколы - это механизм для реализации полиморфизма в Эликсире. Обращение к протоколу доступно для любого типа данных, если этот тип реализует протокол. давайте взглянем на пример.

В Эликсире есть два способа проверить, сколько экземпляров находится в структуре данных: `length` и `size`. `length` значит, что информацию нужно вычислить. Например, `length(list)` должен пройтись по всему списку, чтобы вычислить его длину. С другой стороны, `tuple_size(tuple)` и `byte_size(binary)` просто берёт уже известный размер информации в кортеже или бинарных данных.

Даже если у нас есть встроенные в Эликсир функции для определённых типов, который получают размер (такие как `tuple_size/1`), мы могли бы реализовать общий протокол `Size`, для всех структур данных, у которых размер подсчитан заранее.

Определение протокола бы выглядело подобным образом:

```elixir
defprotocol Size do
  @doc "Calculates the size (and not the length!) of a data structure"
  def size(data)
end
```

Протокол `Size` ожидает, что есть функция `size`, которая принимает один аргумент (структуру данных, размер которой мы хотим узнать). Теперь мы можем реализовать этот протокол для структур данных:

```elixir
defimpl Size, for: BitString do
  def size(string), do: byte_size(string)
end

defimpl Size, for: Map do
  def size(map), do: map_size(map)
end

defimpl Size, for: Tuple do
  def size(tuple), do: tuple_size(tuple)
end
```

Мы не применили протокол `Size` для списков, т.к. для них нет предварительно подсчитанной информации о длине, её нужно вычислять (с помощью `length/1`).

Теперь, имея определение и реализацию протокола, мы можем начать использовать его:

```iex
iex> Size.size("foo")
3
iex> Size.size({:ok, "hello"})
2
iex> Size.size(%{label: "some label"})
1
```

Попытка узнать размер типа данных, которые не принимают этот протокол, вызовет ошибку:

```iex
iex> Size.size([1, 2, 3])
** (Protocol.UndefinedError) protocol Size not implemented for [1, 2, 3]
```

Протоколы можно применять для всех типов данных Эликсира:

* `Atom`
* `BitString`
* `Float`
* `Function`
* `Integer`
* `List`
* `Map`
* `PID`
* `Port`
* `Reference`
* `Tuple`


## Протоколы и структуры

The power of Elixir's extensibility comes when protocols and structs are used together.

Мощность расширямости Эликсира особенно хорошо проявляет себя при использовании протоколов и структур вместе.

В [предыдущей главе](/getting-started/structs.html), мы узнали, что хотя структуры являются словарями, они не разделяют реализацию протоколов словарей. Например, [`MapSet`](https://hexdocs.pm/elixir/MapSet.html) (наборы на основе словарей) реализованы как структуры. Давайте попробуем использовать протокол `Size` для `MapSet`:

```iex
iex> Size.size(%{})
0
iex> set = %MapSet{} = MapSet.new
#MapSet<[]>
iex> Size.size(set)
** (Protocol.UndefinedError) protocol Size not implemented for #MapSet<[]>
```

Вместо разделения реализации протокола со словарями, для структур необходима их собственная реализация протоколов. Т.к. размер `MapSet` подсчитан заранее и доступен через `MapSet.size/1`, мы можем добавить его в `Size`:

```elixir
defimpl Size, for: MapSet do
  def size(set), do: MapSet.size(set)
end
```

Если хотите, вы можете добавить свою семантику для размера своих собственных структур. Кроме того, вы можете использовать структуры для создания более сложных типов данных, например, очередей, и реализовать для них все подходящие протоколы, такие как `Enumerable` и, возможно, `Size`.

```elixir
defmodule User do
  defstruct [:name, :age]
end

defimpl Size, for: User do
  def size(_user), do: 2
end
```

## Implementing `Any`

Manually implementing protocols for all types can quickly become repetitive and tedious. In such cases, Elixir provides two options: we can explicitly derive the protocol implementation for our types or automatically implement the protocol for all types. In both cases, we need to implement the protocol for `Any`.

### Deriving

Elixir allows us to derive a protocol implementation based on the `Any` implementation. Let's first implement `Any` as follows:

```elixir
defimpl Size, for: Any do
  def size(_), do: 0
end
```

The implementation above is arguably not a reasonable one. For example, it makes no sense to say that the size of a `PID` or an `Integer` is `0`.

However, should we be fine with the implementation for `Any`, in order to use such implementation we would need to tell our struct to explicitly derive the `Size` protocol:

```elixir
defmodule OtherUser do
  @derive [Size]
  defstruct [:name, :age]
end
```

When deriving, Elixir will implement the `Size` protocol for `OtherUser` based on the implementation provided for `Any`.

### Fallback to `Any`

Another alternative to `@derive` is to explicitly tell the protocol to fallback to `Any` when an implementation cannot be found. This can be achieved by setting `@fallback_to_any` to `true` in the protocol definition:

```elixir
defprotocol Size do
  @fallback_to_any true
  def size(data)
end
```

As we said in the previous section, the implementation of `Size` for `Any` is not one that can apply to any data type. That's one of the reasons why `@fallback_to_any` is an opt-in behaviour. For the majority of protocols, raising an error when a protocol is not implemented is the proper behaviour. That said, assuming we have implemented `Any` as in the previous section:

```elixir
defimpl Size, for: Any do
  def size(_), do: 0
end
```

Now all data types (including structs) that have not implemented the `Size` protocol will be considered to have a size of `0`.

Which technique is best between deriving and falling back to any depends on the use case but, given Elixir developers prefer explicit over implicit, you may see many libraries pushing towards the `@derive` approach.

## Built-in protocols

Elixir ships with some built-in protocols. In previous chapters, we have discussed the `Enum` module which provides many functions that work with any data structure that implements the `Enumerable` protocol:

```iex
iex> Enum.map [1, 2, 3], fn(x) -> x * 2 end
[2, 4, 6]
iex> Enum.reduce 1..3, 0, fn(x, acc) -> x + acc end
6
```
Another useful example is the `String.Chars` protocol, which specifies how to convert a data structure with characters to a string. It's exposed via the `to_string` function:

```iex
iex> to_string :hello
"hello"
```

Notice that string interpolation in Elixir calls the `to_string` function:

```iex
iex> "age: #{25}"
"age: 25"
```

The snippet above only works because numbers implement the `String.Chars` protocol. Passing a tuple, for example, will lead to an error:

```iex
iex> tuple = {1, 2, 3}
{1, 2, 3}
iex> "tuple: #{tuple}"
** (Protocol.UndefinedError) protocol String.Chars not implemented for {1, 2, 3}
```

When there is a need to "print" a more complex data structure, one can use the `inspect` function, based on the `Inspect` protocol:

```iex
iex> "tuple: #{inspect tuple}"
"tuple: {1, 2, 3}"
```

The `Inspect` protocol is the protocol used to transform any data structure into a readable textual representation. This is what tools like IEx use to print results:

```iex
iex> {1, 2, 3}
{1, 2, 3}
iex> %User{}
%User{name: "john", age: 27}
```

Keep in mind that, by convention, whenever the inspected value starts with `#`, it is representing a data structure in non-valid Elixir syntax. This means the inspect protocol is not reversible as information may be lost along the way:

```iex
iex> inspect &(&1+2)
"#Function<6.71889879/1 in :erl_eval.expr/5>"
```

There are other protocols in Elixir but this covers the most common ones.

## Protocol consolidation

When working with Elixir projects, using the Mix build tool, you may see the output as follows:

```
Consolidated String.Chars
Consolidated Collectable
Consolidated List.Chars
Consolidated IEx.Info
Consolidated Enumerable
Consolidated Inspect
```

Those are all protocols that ship with Elixir and they are being consolidated. Because a protocol can dispatch to any data type, the protocol must check on every call if an implementation for the given type exists. This may be expensive.

However, after our project is compiled using a tool like Mix, we know all modules that have been defined, including protocols and their implementations. This way, the protocol can be consolidated into a very simple and fast dispatch module.

From Elixir v1.2, protocol consolidation happens automatically for all projects. We will build our own project in the ***Mix and OTP guide***.
