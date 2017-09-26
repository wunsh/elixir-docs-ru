---
title: Структуры
next_page: protocols
prev_page: module-attributes
---

## Определение структур

Для определения структуры в `elixir` используется  конструкция `defstruct`:

```elixir
iex> defmodule User do
...>   defstruct name: "John", age: 27
...> end
```

Список ключевых слов, используемый при определении defstruct, определяет, какие поля будет иметь структура вместе со значениями по умолчанию.

Структуры берут имя модуля, в котором они определены. В приведенном выше примере мы определили структуру с именем `User`.

Теперь мы можем создавать `User` структуры, используя синтаксис, аналогичный тому, который используется для создания словарей:
```elixir
iex> %User{}
%User{age: 27, name: "John"}
iex> %User{name: "Meg"}
%User{age: 27, name: "Meg"}
```
Структуры предоставляют гарантии времени компиляции (compile-time guarantees),  только поля определенные через, `defstruct` будут разрешены к существованию в структуре:
```elixir
iex> %User{oops: :field}
** (KeyError) key :oops not found in: %User{age: 27, name: "John"}
```
## Доступ и обновление структур

Когда мы обсуждали словари, мы показали, как мы можем получить доступ и обновить поля словаря. Те же самые методы (и тот же синтаксис) применимы и к структурам:
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
При использовании синтаксиса `update ( | )`, виртуальная машина(VM) знает, что в структуру не будут добавлены новые ключи, позволяющие словарям делиться своей структурой в памяти. В приведенном выше примере обе john и meg совместно используют одну и ту же ключевую структуру в памяти.

Структуры также могут использоваться при сопоставлении шаблонов, как для сопоставления значений конкретных клавиш, так и для обеспечения того, что значение соответствия является структурой того же типа, что и согласованное значение.
```elixir
iex> %User{name: name} = john
%User{age: 27, name: "John"}
iex> name
"John"
iex> %User{} = %{}
** (MatchError) no match of right hand side value: %{}
```
## Структуры - это обычные словари внутри

В приведенном выше примере сопоставление шаблонов работает, потому что подструктурами находятся обычные словари с фиксированным набором полей. В качестве карт структура хранит «специальное» поле с именем, `__struct__` которое содержит имя структуры:
```elixir
iex> is_map(john)
true
iex> john.__struct__
User
```
Обратите внимание, что мы ссылались на структуры как на обычные словари, потому что ни один из протоколов, реализованных для словарей, не доступен для структур. Например, вы не можете ни перечислить, ни получить доступ к структуре:
```elixir
iex> john = %User{}
%User{age: 27, name: "John"}
iex> john[:name]
** (UndefinedFunctionError) function User.fetch/2 is undefined (User does not implement the Access behaviour)
             User.fetch(%User{age: 27, name: "John"}, :name)
iex> Enum.each john, fn({field, value}) -> IO.puts(value) end
** (Protocol.UndefinedError) protocol Enumerable not implemented for %User{age: 27, name: "John"}
```
Однако, поскольку структуры являются только картами, они работают с функциями из модуля `Map`:
```elixir
iex> kurt = Map.put(%User{}, :name, "Kurt")
%User{age: 27, name: "Kurt"}
iex> Map.merge(kurt, %User{name: "Takashi"})
%User{age: 27, name: "Takashi"}
iex> Map.keys(john)
[:__struct__, :age, :name]
```
Структуры наряду с протоколами обеспечивают одну из наиболее важных функций для разработчиков `Elixir`: полиморфизм данных. Это мы рассмотрим в следующей главе.

## Значения по умолчанию и требуемые ключи

Если вы не укажете значение ключа по умолчанию при определении структуры,  предполагается `nil`:
```elixir
iex> defmodule Product do
...>   defstruct [:name]
...> end
iex> %Product{}
%Product{name: nil}
```
Вы также можете указать, что определенные ключи должны быть указаны при создании структуры:
```elixir
iex> defmodule Car do
...>   @enforce_keys [:make]
...>   defstruct [:model, :make]
...> end
iex> %Car{}
** (ArgumentError) the following keys must also be given when building struct Car: [:make]
    expanding struct: Car.__struct__/1
```
