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

## Named functions

Inside a module, we can define functions with `def/2` and private functions with `defp/2`. A function defined with `def/2` can be invoked from other modules while a private function can only be invoked locally.

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

Function declarations also support guards and multiple clauses. If a function has several clauses, Elixir will try each clause until it finds one that matches. Here is an implementation of a function that checks if the given number is zero or not:

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

Giving an argument that does not match any of the clauses raises an error.

Similar to constructs like `if`, named functions support both `do:` and `do`/`end` block syntax, as [we learned `do`/`end` is a convenient syntax for the keyword list format](/getting-started/case-cond-and-if.html#doend-blocks). For example, we can edit `math.exs` to look like this:

```elixir
defmodule Math do
  def zero?(0), do: true
  def zero?(x) when is_integer(x), do: false
end
```

And it will provide the same behaviour. You may use `do:` for one-liners but always use `do`/`end` for functions spanning multiple lines.

## Function capturing

Throughout this tutorial, we have been using the notation `name/arity` to refer to functions. It happens that this notation can actually be used to retrieve a named function as a function type. Start `iex`, running the `math.exs` file defined above:

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

Remember Elixir makes a distinction between anonymous functions and named functions, where the former must be invoked with a dot (`.`) between the variable name and parentheses. The capture operator bridges this gap by allowing named functions to be assigned to variables and passed as arguments in the same way we assign, invoke and pass anonymous functions.

Local or imported functions, like `is_function/1`, can be captured without the module:

```iex
iex> &is_function/1
&:erlang.is_function/1
iex> (&is_function/1).(fun)
true
```

Note the capture syntax can also be used as a shortcut for creating functions:

```iex
iex> fun = &(&1 + 1)
#Function<6.71889879/1 in :erl_eval.expr/5>
iex> fun.(1)
2
```

The `&1` represents the first argument passed into the function. `&(&1+1)` above is exactly the same as `fn x -> x + 1 end`. The syntax above is useful for short function definitions.

If you want to capture a function from a module, you can do `&Module.function()`:

```iex
iex> fun = &List.flatten(&1, &2)
&List.flatten/2
iex> fun.([1, [[2], 3]], [4, 5])
[1, 2, 3, 4, 5]
```

`&List.flatten(&1, &2)` is the same as writing `fn(list, tail) -> List.flatten(list, tail) end` which in this case is equivalent to `&List.flatten/2`. You can read more about the capture operator `&` in [the `Kernel.SpecialForms` documentation](https://hexdocs.pm/elixir/Kernel.SpecialForms.html#&/1).

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