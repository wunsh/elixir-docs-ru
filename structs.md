---
title: Структуры
next_page: protocols
prev_page: module-attributes
---

## Определение структур

Для определения структуры в Эликсире используется  конструкция `defstruct`:

```elixir
iex> defmodule User do
...>   defstruct name: "John", age: 27
...> end
```

Список ключевых слов, используемый при определении `defstruct`, определяет какие поля будет иметь структура вместе со значениями по умолчанию.

Структуры берут имя модуля, в котором они определены. В приведенном выше примере мы определили структуру с именем `User`.

Теперь мы можем создавать структуру `User` с помощью синтаксиса, аналогичного тому, который используется для создания словарей:

```elixir
iex> %User{}
%User{age: 27, name: "John"}

iex> %User{name: "Meg"}
%User{age: 27, name: "Meg"}
```

Структуры предоставляют гарантии *времени компиляции* того, что только определенные через `defstruct` поля будут разрешены к существованию в структуре:

```elixir
iex> %User{oops: :field}
** (KeyError) key :oops not found in: %User{age: 27, name: "John"}
```
## Доступ к данных структуры и их обновление

При обсуждении словарей мы показывали получение доступа к данным словаря и их обновление. Такой же подход (и такой же синтаксис) применим и к структурам:

```elixir
iex> john = %User{}
%User{age: 27, name: "John"}

iex> john.name
"John"

iex> meg = %{john | name: "Meg"}
%User{age: 27, name: "Meg"}

iex> %{meg | oops: :field}
** (KeyError) key :oops not found in: %User{age: 27, name: "Meg"}
```
При использовании синтаксиса обновления данных (`|`) виртуальная машина знает, что в структуру не будет добавлено новых ключей, позволяя тем самым словарям использовать общую память для хранения своей ключевой структуры. В приведенном выше примере обе структуры `john` и `meg` совместно используют одну и ту же ключевую структуру в памяти.

Структуры также могут использоваться при сопоставлении с образцом, как для сопоставления значений конкретных ключей, так и для проверки того, что сопоставляемое значение является структурой того же типа, что и образец.

```elixir
iex> %User{name: name} = john
%User{age: 27, name: "John"}

iex> name
"John"

iex> %User{} = %{}
** (MatchError) no match of right hand side value: %{}
```

## Структуры – это голые словари

В приведенном выше примере сопоставление c образцом работает благодаря тому, что под капотом структуры представляют собой голые словари с фиксированным набором полей. Вместе со словарём в структуре хранится «специальное» поле `__struct__`, которое содержит название структуры:

```elixir
iex> is_map(john)
true

iex> john.__struct__
User
```
Обратите внимание, что мы структуры являются **голыми словарями**, потому что ни один из протоколов, реализованных для словарей, не доступен для структур. Например, вы не можете ни перечислить значения, ни получить прямой доступ к значениям:

```elixir
iex> john = %User{}
%User{age: 27, name: "John"}

iex> john[:name]
** (UndefinedFunctionError) function User.fetch/2 is undefined (User does not implement the Access behaviour)

User.fetch(%User{age: 27, name: "John"}, :name)
iex> Enum.each john, fn({field, value}) -> IO.puts(value) end
** (Protocol.UndefinedError) protocol Enumerable not implemented for %User{age: 27, name: "John"}
```

Тем не менее, поскольку структуры являются всего лишь словарями, они работают с функциями из модуля `Map`:

```elixir
iex> kurt = Map.put(%User{}, :name, "Kurt")
%User{age: 27, name: "Kurt"}

iex> Map.merge(kurt, %User{name: "Takashi"})
%User{age: 27, name: "Takashi"}

iex> Map.keys(john)
[:__struct__, :age, :name]
```

Структуры, наряду с протоколами, обеспечивают одну из наиболее важных функций для эликсирщиков: полиморфизм данных. Мы поговорим об этом в [следующей главе](/docs/protocols.html).

## Значения по умолчанию и обязательные ключи

Если вы не укажете значение ключа по умолчанию при определении структуры, предполагается `nil`:

```elixir
iex> defmodule Product do
...>   defstruct [:name]
...> end

iex> %Product{}
%Product{name: nil}
```

Вы также можете обозначить, что определенные ключи должны быть обязательно указаны при создании структуры:

```elixir
iex> defmodule Car do
...>   @enforce_keys [:make]
...>   defstruct [:model, :make]
...> end

iex> %Car{}
** (ArgumentError) the following keys must also be given when building struct Car: [:make]
    expanding struct: Car.__struct__/1
```
