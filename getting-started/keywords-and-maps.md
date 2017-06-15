# Списки с ключевыми словами и мэпы

До сих пор мы не говорили ни о каких ассоциативных структурах данных, такие структуры позволяют ассоциировать некоторое значение (или несколько значений) с ключом.

В Elixir у нас есть два вида ассоциативных структур данных: списки с ключевыми словами и мэпы. Самое время познакомиться с ними!

## Списки с ключевыми словами

Во многих функциональных языках программирования распространено использование массивов из кортежей, которые состоят из двух элементов, чтобы представить структуру из пар ключ-значение. В Elixir, когда у нас есть список кортежей, и первый элемент кортежа (ключ) - это атом, мы называем это списком с ключевыми словами (keyword list):

```iex
iex> list = [{:a, 1}, {:b, 2}]
[a: 1, b: 2]
iex> list == [a: 1, b: 2]
true
```

Как вы можете увидеть выше, Elixir поддерживает специальный синтаксис для объявления таких списков: `[key: value]`. Под капотом это интерпретируется как список кортежей выше. Т.к. списки с ключевыми словами - объекты типа List, мы можем делать с ними все операции, доступные для списков. Например, можно добавить новое значение, используя `++`:

```iex
iex> list ++ [c: 3]
[a: 1, b: 2, c: 3]
iex> [a: 0] ++ list
[a: 0, a: 1, b: 2]
```

Обратите внимание, что при поиске будут возвращаться значения, стоящие ближе к началу списка:

```iex
iex> new_list = [a: 0] ++ list
[a: 0, a: 1, b: 2]
iex> new_list[:a]
0
```

Списки с ключевыми словами важны, потому что они имеют три особые характеристики:

  * Ключи должны быть атомами.
  * Ключи отсортированы так, как их задал разработчик.
  * Ключи могут быть добавлены более одного раза.

Например, [библиотека Ecto](https://github.com/elixir-lang/ecto) использует эти особенности для предоставления элегантного DSL для написания запросов к базам данных:

```elixir
query = from w in Weather,
      where: w.prcp > 0,
      where: w.temp < 20,
     select: w
```

Эти характеристики являются причиной, по которой списки с ключами являются стандартным механизмом передачи опций в функции Elixir. В главе 5, когда мы обсуждали макрос `if/2`, мы упоминали, что поддерживается следующий синтаксис:

```iex
iex> if false, do: :this, else: :that
:that
```

Пары `do:` и `else:` - это списки с ключевыми словами! Фактически, вызов выше соответствует этому:

```iex
iex> if(false, [do: :this, else: :that])
:that
```

А это, как мы видели выше, тоже самое, что:

```iex
iex> if(false, [{:do, :this}, {:else, :that}])
:that
```

Когда список с ключами является последним аргументом функции, квадратные скобки не обязательны.

Хотя мы можем сравнивать по шаблону списки с ключами, это редко делается на практике, потому что предусматривает соответствие количества элементов и их порядка:

```iex
iex> [a: a] = [a: 1]
[a: 1]
iex> a
1
iex> [a: a] = [a: 1, b: 2]
** (MatchError) no match of right hand side value: [a: 1, b: 2]
iex> [b: b, a: a] = [a: 1, b: 2]
** (MatchError) no match of right hand side value: [a: 1, b: 2]
```

Для манипуляций со списками с ключевыми словами Elixir предоставляет [модуль `Keyword`](https://hexdocs.pm/elixir/Keyword.html). Помните, что списки с ключами - это просто списки, с такими же линейными характеристиками производительности. Чем длиннее список, тем дольше будет поиск по ключу, подсчёт количества элементов и т.д. По этой причине такие списки используются в Elixir главным образом для передачи дополнительных значений. Если вам нужно хранить много элементов или исключить дублирование ключей, вам следует использовать мэпы.

## Maps

Whenever you need a key-value store, maps are the "go to" data structure in Elixir. A map is created using the `%{}` syntax:

```iex
iex> map = %{:a => 1, 2 => :b}
%{2 => :b, :a => 1}
iex> map[:a]
1
iex> map[2]
:b
iex> map[:c]
nil
```

Compared to keyword lists, we can already see two differences:

  * Maps allow any value as a key.
  * Maps' keys do not follow any ordering.

In contrast to keyword lists, maps are very useful with pattern matching. When a map is used in a pattern, it will always match on a subset of the given value:

```iex
iex> %{} = %{:a => 1, 2 => :b}
%{2 => :b, :a => 1}
iex> %{:a => a} = %{:a => 1, 2 => :b}
%{2 => :b, :a => 1}
iex> a
1
iex> %{:c => c} = %{:a => 1, 2 => :b}
** (MatchError) no match of right hand side value: %{2 => :b, :a => 1}
```

As shown above, a map matches as long as the keys in the pattern exist in the given map. Therefore, an empty map matches all maps.

Variables can be used when accessing, matching and adding map keys:

```iex
iex> n = 1
1
iex> map = %{n => :one}
%{1 => :one}
iex> map[n]
:one
iex> %{^n => :one} = %{1 => :one, 2 => :two, 3 => :three}
%{1 => :one, 2 => :two, 3 => :three}
```

[The `Map` module](https://hexdocs.pm/elixir/Map.html) provides a very similar API to the `Keyword` module with convenience functions to manipulate maps:

```iex
iex> Map.get(%{:a => 1, 2 => :b}, :a)
1
iex> Map.put(%{:a => 1, 2 => :b}, :c, 3)
%{2 => :b, :a => 1, :c => 3}
iex> Map.to_list(%{:a => 1, 2 => :b})
[{2, :b}, {:a, 1}]
```

Maps have the following syntax for updating a key's value:

```iex
iex> map = %{:a => 1, 2 => :b}
%{2 => :b, :a => 1}

iex> %{map | 2 => "two"}
%{2 => "two", :a => 1}
iex> %{map | :c => 3}
** (KeyError) key :c not found in: %{2 => :b, :a => 1}
```

The syntax above requires the given key to exist. It cannot be used to add new keys. For example, using it with the `:c` key failed because there is no `:c` in the map.

When all the keys in a map are atoms, you can use the keyword syntax for convenience:

```iex
iex> map = %{a: 1, b: 2}
%{a: 1, b: 2}
```

Another interesting property of maps is that they provide their own syntax for accessing atom keys:

```iex
iex> map = %{:a => 1, 2 => :b}
%{2 => :b, :a => 1}

iex> map.a
1
iex> map.c
** (KeyError) key :c not found in: %{2 => :b, :a => 1}
```

Elixir developers typically prefer to use the `map.field` syntax and pattern matching instead of the functions in the `Map` module when working with maps because they lead to an assertive style of programming. [This blog post](http://blog.plataformatec.com.br/2014/09/writing-assertive-code-with-elixir/) provides insight and examples on how you get more concise and faster software by writing assertive code in Elixir.

> Note: Maps were recently introduced into the Erlang <abbr title="Virtual Machine">VM</abbr> and only from Elixir v1.2 they are capable of holding millions of keys efficiently. Therefore, if you are working with previous Elixir versions (v1.0 or v1.1) and you need to support at least hundreds of keys, you may consider using [the `HashDict` module](https://hexdocs.pm/elixir/HashDict.html).

## Nested data structures

Often we will have maps inside maps, or even keywords lists inside maps, and so forth. Elixir provides conveniences for manipulating nested data structures via the `put_in/2`, `update_in/2` and other macros giving the same conveniences you would find in imperative languages while keeping the immutable properties of the language.

Imagine you have the following structure:

```iex
iex> users = [
  john: %{name: "John", age: 27, languages: ["Erlang", "Ruby", "Elixir"]},
  mary: %{name: "Mary", age: 29, languages: ["Elixir", "F#", "Clojure"]}
]
[john: %{age: 27, languages: ["Erlang", "Ruby", "Elixir"], name: "John"},
 mary: %{age: 29, languages: ["Elixir", "F#", "Clojure"], name: "Mary"}]
```

We have a keyword list of users where each value is a map containing the name, age and a list of programming languages each user likes. If we wanted to access the age for john, we could write:

```iex
iex> users[:john].age
27
```

It happens we can also use this same syntax for updating the value:

```iex
iex> users = put_in users[:john].age, 31
[john: %{age: 31, languages: ["Erlang", "Ruby", "Elixir"], name: "John"},
 mary: %{age: 29, languages: ["Elixir", "F#", "Clojure"], name: "Mary"}]
```

The `update_in/2` macro is similar but allows us to pass a function that controls how the value changes. For example, let's remove "Clojure" from Mary's list of languages:

```iex
iex> users = update_in users[:mary].languages, fn languages -> List.delete(languages, "Clojure") end
[john: %{age: 31, languages: ["Erlang", "Ruby", "Elixir"], name: "John"},
 mary: %{age: 29, languages: ["Elixir", "F#"], name: "Mary"}]
```

There is more to learn about `put_in/2` and `update_in/2`, including the `get_and_update_in/2` that allows us to extract a value and update the data structure at once. There are also `put_in/3`, `update_in/3` and `get_and_update_in/3` which allow dynamic access into the data structure. [Check their respective documentation in the `Kernel` module for more information](https://hexdocs.pm/elixir/Kernel.html).

This concludes our introduction to associative data structures in Elixir. You will find out that, given keyword lists and maps, you will always have the right tool to tackle problems that require associative data structures in Elixir.