# Модули и функции

В Elixir мы объединяем некоторые функции в модули. Мы уже использовали много разных модулей в предыдущих главах, например [модуль `String`](https://hexdocs.pm/elixir/String.html):

```iex
iex> String.length("hello")
5
```

Чтобы создать свой модуль в Elixir, мы используем макрос `defmodule`. Мы используем макрос `def` для объявления функций в этом модуле:

```iex
iex> defmodule Math do
...>   def sum(a, b) do
...>     a + b
...>   end
...> end

iex> Math.sum(1, 2)
3
```

В следующих секциях наши примеры будут больше, и будет не очень удобно набирать их в консоли. Самое время научиться компилировать Elixir код и запусть скрипты на Elixir.

## Компиляция

Как правило, очень удобно писать модули в файлы, так их можно скомпилировать и использовать повторно. Допустим, у нас есть файл с именем `math.ex` со следующим содержимым:

```elixir
defmodule Math do
  def sum(a, b) do
    a + b
  end
end
```

Этот файл может быть скомпилирован, используя `elixirc`:

```bash
$ elixirc math.ex
```

Это создаст файл `Elixir.Math.beam`, содержащий байткод описанного модуля. Если мы снова запустим `iex` снова, модуль будет доступен (`iex` должен быть запущен в той же директории, где лежит файл с байткодом):

```iex
iex> Math.sum(1, 2)
3
```

Проекты на Elixir обычно организованы в три директории:

* ebin - содержит скомпилированный байткод
* lib - содержит elixir код (как правило файлы `.ex`)
* test - содержит тесты (обычно файлы `.exs`)

Во время работы над реальными проектам, инструмент сборки `mix` будет отвечать за компиляцию и настройки правильных путей вместо вас. Для образовательных целей, Elixir также поддерживает скриптовый режим, более гибкий и без генерации каких-либо скомпилированных артефактов (корректный перевод artifacts?).

## Скриптовый режим

В дополнение к расширению файлов `.ex`, в Elixir также поддерживаются `.exs` файлы для написания скриптов. C точки зрения Elixir такие файлы идентичны, разница лишь в их назначении. `.ex` файлы будут скомпилированы, в то время как `.exs` файлы используются для скриптов. При исполнении, оба расширения компилируются и загружаются свои модули в память, но только `.ex` файлы записывают свой байткод на диск в формате `.beam`.

Например, мы можем создать файл с названием `math.exs`:

```elixir
defmodule Math do
  def sum(a, b) do
    a + b
  end
end

IO.puts Math.sum(1, 2)
```

И выполнить его:

```bash
$ elixir math.exs
```

Файл скомпилируется в оперативную память и выполнится, напечатает результат "3". Файл байткода не будет создан. В следующих примерах мы рекомендуем вам писать ваш код в скриптовые файлы и выполнять их как показано выше.

## Именованные функции

Внутри модуля мы можем определить функцию при помощи `def/2` и приватную функцию, использовав `defp/2`. Функция, определённая через `def/2` может быть вызвана из другого модуля, а приватная функция может быть вызвана только локально.

```elixir
defmodule Math do
  def sum(a, b) do
    do_sum(a, b)
  end

  defp do_sum(a, b) do
    a + b
  end
end

IO.puts Math.sum(1, 2)    #=> 3
IO.puts Math.do_sum(1, 2) #=> ** (UndefinedFunctionError)
```

Определения функция также поддерживают безопасные ограничения (guards) и множественные варианты аргументов. Elixir будет проверять каждый вариант, пока не найдёт подходящий. Вот реализация функции, которая проверяет, является ли переданное ей число нулём:

```elixir
defmodule Math do
  def zero?(0) do
    true
  end

  def zero?(x) when is_integer(x) do
    false
  end
end

IO.puts Math.zero?(0)         #=> true
IO.puts Math.zero?(1)         #=> false
IO.puts Math.zero?([1, 2, 3]) #=> ** (FunctionClauseError)
IO.puts Math.zero?(0.0)       #=> ** (FunctionClauseError)
```

Передача аргумента, который не подходит ни одному из вариантов, вызовет ошибку.

Аналогично конструкциям, вроде `if`, именованные функции поддерживают как синтаксис блоков `do:`, так и `do`/`end`, который [мы изучили вместе со списками с ключевыми словами](/getting-started/case-cond-and-if.html#doend-blocks). Например, мы можем привести `math.exs` к следующему виду:

```elixir
defmodule Math do
  def zero?(0), do: true
  def zero?(x) when is_integer(x), do: false
end
```

Такой вариант даст нам аналогичное поведение. Вы можете использовать `do:` для записи в одну строку, но для многострочного кода всегда нужны `do`/`end`.

## Отлов функций

На протяжении этого руководства мы использовали нотацию `имя_функции/арность` для обозначений функций. Такая нотация также может быть использована для получения именованной функции в качестве типа "функция". Запустите `iex`, открыв описанный выше файл `math.exs`:

```bash
$ iex math.exs
```

```iex
iex> Math.zero?(0)
true
iex> fun = &Math.zero?/1
&Math.zero?/1
iex> is_function(fun)
true
iex> fun.(0)
true
```

Помните, что в Elixir есть разница между анонимными и именованными функциями. В первых для вызова должна быть указана точка (`.`) между именем переменной и скобками. Оператор захвата (`&`, capture operator) позволяет именованным функциям быть привязанным к переменным и переданным в качестве арументам так же, как мы привязываем, исполняем и передаём анонимные функции.

Локальные или импортированные функции, такие как `is_function/1`, могут быть использованы без модуля:

```iex
iex> &is_function/1
&:erlang.is_function/1
iex> (&is_function/1).(fun)
true
```

Обратите внимание, что синтаксис захвата может быть также использован в качестве сокращения для создания функций:

```iex
iex> fun = &(&1 + 1)
#Function<6.71889879/1 in :erl_eval.expr/5>
iex> fun.(1)
2
```

`&1` воспринимается как первый аргумент, переданный в функцию. `&(&1+1)` выше, то же самое, что `fn x -> x + 1 end`. Синтаксис выше полезен для объявления коротких функций.

Если вы хотите захватить функцию из модуля, вы можете использовать `&Module.function()`:

```iex
iex> fun = &List.flatten(&1, &2)
&List.flatten/2
iex> fun.([1, [[2], 3]], [4, 5])
[1, 2, 3, 4, 5]
```

`&List.flatten(&1, &2)` аналогично `fn(list, tail) -> List.flatten(list, tail) end`, а это в данном случае равноценно `&List.flatten/2`. Вы можете получить больше информации об операторе `&` в [документации по `Kernel.SpecialForms`](https://hexdocs.pm/elixir/Kernel.SpecialForms.html#&/1).

## Default arguments

Named functions in Elixir also support default arguments:

```elixir
defmodule Concat do
  def join(a, b, sep \\ " ") do
    a <> sep <> b
  end
end

IO.puts Concat.join("Hello", "world")      #=> Hello world
IO.puts Concat.join("Hello", "world", "_") #=> Hello_world
```

Any expression is allowed to serve as a default value, but it won't be evaluated during the function definition. Every time the function is invoked and any of its default values have to be used, the expression for that default value will be evaluated:

```elixir
defmodule DefaultTest do
  def dowork(x \\ "hello") do
    x
  end
end
```

```iex
iex> DefaultTest.dowork
"hello"
iex> DefaultTest.dowork 123
123
iex> DefaultTest.dowork
"hello"
```

If a function with default values has multiple clauses, it is required to create a function head (without an actual body) for declaring defaults:

```elixir
defmodule Concat do
  def join(a, b \\ nil, sep \\ " ")

  def join(a, b, _sep) when is_nil(b) do
    a
  end

  def join(a, b, sep) do
    a <> sep <> b
  end
end

IO.puts Concat.join("Hello", "world")      #=> Hello world
IO.puts Concat.join("Hello", "world", "_") #=> Hello_world
IO.puts Concat.join("Hello")               #=> Hello
```

When using default values, one must be careful to avoid overlapping function definitions. Consider the following example:

```elixir
defmodule Concat do
  def join(a, b) do
    IO.puts "***First join"
    a <> b
  end

  def join(a, b, sep \\ " ") do
    IO.puts "***Second join"
    a <> sep <> b
  end
end
```

If we save the code above in a file named "concat.ex" and compile it, Elixir will emit the following warning:

    warning: this clause cannot match because a previous clause at line 2 always matches

The compiler is telling us that invoking the `join` function with two arguments will always choose the first definition of `join` whereas the second one will only be invoked when three arguments are passed:

```bash
$ iex concat.exs
```

```iex
iex> Concat.join "Hello", "world"
***First join
"Helloworld"
```

```iex
iex> Concat.join "Hello", "world", "_"
***Second join
"Hello_world"
```

This finishes our short introduction to modules. In the next chapters, we will learn how to use named functions for recursion, explore Elixir lexical directives that can be used for importing functions from other modules and discuss module attributes.