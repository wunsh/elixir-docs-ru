---
title: Макросы
---

# {{ page.title }}

## Введение

Несмотря на то, что Эликсир пытается обеспечить безопасную среду для макросов, основная ответственность за написание чистого кода с помощью макросов ложится на разработчиков. Писать макросы сложнее, чем обычные функции на Эликсире, и соответсвенно их не стоит использовать, когда в них нет нужды. Так что пишите и применяйте макросы со всей ответственностью.

Эликсир и так предоставляет механизмы для написания обычного кода простым и понятным способом, используя встроенные структуры данных и функции. Следовательно, макросы должны использоваться исключительно в крайнем случае. Помните, что **явное лучше, чем неявное. Чистый код лучше, чем сокращённый**

## Наш первый макрос

Макросы в Эликсире определяются через `defmacro/2`.

> В этой главе мы будет использовать файлы вместо запуска примеров кода в IEx. Всё потому, что эти примеры будут всего лишь в несколько строк кода, поэтому их ввод в IEx будет контрпродуктивным. Следовательно, вы должны иметь возможность проверять эти примеры, записывая их в файл `macros.exs` и запускать с помощью `elixir macros.exs` или `iex macros.exs`.

Чтобы лучше понять, как работают макросы, давайте сперва создадим новый модуль, в котором реализуем условный оператор `unless`, который делает противоположный `if`, как макросом, так и функцией: 

```elixir
defmodule Unless do
  def fun_unless(clause, do: expression) do
    if(!clause, do: expression)
  end

  defmacro macro_unless(clause, do: expression) do
    quote do
      if(!unquote(clause), do: unquote(expression))
    end
  end
end
```

Функция получает аргументы и передаёт их в `if`. Однако, как мы узнали в [предыдущей главе](meta / quote-and-unquote.md), макрос будет получать маскирующие выражения, вводить их в quote и, в конце, возвращать другое маскирующее выражение. 

Давайте начнём с `iex`, используя модуль выше:

```bash
$ iex macros.exs
```

И поиграем с теми определениями:

```iex
iex> require Unless
iex> Unless.macro_unless true, do: IO.puts "this should never be printed"
nil
iex> Unless.fun_unless true, do: IO.puts "this should never be printed"
"this should never be printed"
nil
```

Обратите внимание на то, что в нашей реализации макроса предложение не было возвращено, несмотря на то, что оно вывелось при реализации нашей функции. Это обусловленно тем, что аргументы вызова функции вычисляются непосредственно перед вызовом самой функции. Однако макросы не обращают внимание на свои отдельно взятые аргументы. Вместо этого, они получают аргументы в виде маскирующих выражений, которые затем преобразуются в другие маскирующие выражения. Таким образом, нам придётся переписать наш оператор `unless` в макросе на скрытый `if`.

Другими словами, при вызове:

```elixir
Unless.macro_unless true, do: IO.puts "this should never be printed"
```

Наш макрос `macro_unless` получил следующее:

```erlang
macro_unless(true, [do: {{:., [], [{:__aliases__, [alias: false], [:IO]}, :puts]}, [], ["this should never be printed"]}])
```

И затем он вернул маскирующее выражение следующим образом:

```erlang
{:if, [],
 [{:!, [], [true]},
  [do: {{:., [],
     [{:__aliases__,
       [], [:IO]},
      :puts]}, [], ["this should never be printed"]}]]}
```

На самом деле, мы можем проверить это, используя `Macro.expand_once/2`:

```iex
iex> expr = quote do: Unless.macro_unless(true, do: IO.puts "this should never be printed")
iex> res  = Macro.expand_once(expr, __ENV__)
iex> IO.puts Macro.to_string(res)
if(!true) do
  IO.puts("this should never be printed")
end
:ok
```

`Macro.expand_once/2` получает маскирующее выражение и расширяет его в соответствии с текущей средой. В этом случае он расширил/вызвал макрос `Unless.macro_unless/2` и вернул его результат. Затем мы перешли к преобразованию возвращаемого маскирующего выражения в строку и напечатали его (мы затронем `__ENV__` чуть позже в этой главе).

Вот что такое макросы. Они про получение маскирующих выражений и преобразование их во что-то иное. Фактически, `unless/2` в Эликсире реализуется как макрос:

```elixir
defmacro unless(clause, do: expression) do
  quote do
    if(!unquote(clause), do: unquote(expression))
  end
end
```

Конструкции, вроде `unless/2`, `defmacro/2`, `def/2`, `defprotocol/2` и многие другие, использующиеся в этом руководстве с самого начала, реализованы на чистом Эликсире, часто в качестве макроса. Это означает, что конструкторы, использующиеся для создания языка, могут использоваться разработчиками для расширения предметно-ориентированного языка, над которым они работают.

Мы может определить любые необходимые нам функции и макросы, в том числе и те, которые переопределяют встроенные определения, предоставляемые самим Эликсиром. Единственным исключением являются специальные формы Эликсира, которые не реализованы в нём и поэтому не могут быть переопределёнными априори [полный список специальных форм доступен в `Kernel.SpecialForms`](https://hexdocs.pm/elixir/).

## Macros hygiene

Elixir macros have late resolution. This guarantees that a variable defined inside a quote won’t conflict with a variable defined in the context where that macro is expanded. For example:

```elixir
defmodule Hygiene do
  defmacro no_interference do
    quote do: a = 1
  end
end

defmodule HygieneTest do
  def go do
    require Hygiene
    a = 13
    Hygiene.no_interference
    a
  end
end

HygieneTest.go
# => 13
```

In the example above, even though the macro injects `a = 1`, it does not affect the variable `a` defined by the `go` function. If a macro wants to explicitly affect the context, it can use `var!`:

```elixir
defmodule Hygiene do
  defmacro interference do
    quote do: var!(a) = 1
  end
end

defmodule HygieneTest do
  def go do
    require Hygiene
    a = 13
    Hygiene.interference
    a
  end
end

HygieneTest.go
# => 1
```

Variable hygiene only works because Elixir annotates variables with their context. For example, a variable `x` defined on line 3 of a module would be represented as:

```elixir
{:x, [line: 3], nil}
```

However, a quoted variable is represented as:

```elixir
defmodule Sample do
  def quoted do
    quote do: x
  end
end

Sample.quoted #=> {:x, [line: 3], Sample}
```

Notice that the third element in the quoted variable is the atom `Sample`, instead of `nil`, which marks the variable as coming from the `Sample` module. Therefore, Elixir considers these two variables as coming from different contexts and handles them accordingly.

Elixir provides similar mechanisms for imports and aliases too. This guarantees that a macro will behave as specified by its source module rather than conflicting with the target module where the macro is expanded. Hygiene can be bypassed under specific situations by using macros like `var!/2` and `alias!/2`, although one must be careful when using those as they directly change the user environment.

Sometimes variable names might be dynamically created. In such cases, `Macro.var/2` can be used to define new variables:

```elixir
defmodule Sample do
  defmacro initialize_to_char_count(variables) do
    Enum.map variables, fn(name) ->
      var = Macro.var(name, nil)
      length = name |> Atom.to_string |> String.length
      quote do
        unquote(var) = unquote(length)
      end
    end
  end

  def run do
    initialize_to_char_count [:red, :green, :yellow]
    [red, green, yellow]
  end
end

> Sample.run #=> [3, 5, 6]
```

Take note of the second argument to `Macro.var/2`. This is the context being used and will determine hygiene as described in the next section.

## The environment

When calling `Macro.expand_once/2` earlier in this chapter, we used the special form `__ENV__`.

`__ENV__` returns an instance of the `Macro.Env` struct which contains useful information about the compilation environment, including the current module, file, and line, all variables defined in the current scope, as well as imports, requires and so on:

```iex
iex> __ENV__.module
nil
iex> __ENV__.file
"iex"
iex> __ENV__.requires
[IEx.Helpers, Kernel, Kernel.Typespec]
iex> require Integer
nil
iex> __ENV__.requires
[IEx.Helpers, Integer, Kernel, Kernel.Typespec]
```

Many of the functions in the `Macro` module expect an environment. You can read more about these functions in [the docs for the `Macro` module](https://hexdocs.pm/elixir/) and learn more about the compilation environment in the [docs for `Macro.Env`](https://hexdocs.pm/elixir/).

## Private macros

Elixir also supports private macros via `defmacrop`. As private functions, these macros are only available inside the module that defines them, and only at compilation time.

It is important that a macro is defined before its usage. Failing to define a macro before its invocation will raise an error at runtime, since the macro won’t be expanded and will be translated to a function call:

```iex
iex> defmodule Sample do
...>  def four, do: two + two
...>  defmacrop two, do: 2
...> end
** (CompileError) iex:2: function two/0 undefined
```

## Write macros responsibly

Macros are a powerful construct and Elixir provides many mechanisms to ensure they are used responsibly.

* Macros are hygienic: by default, variables defined inside a macro are not going to affect the user code. Furthermore, function calls and aliases available in the macro context are not going to leak into the user context.

* Macros are lexical: it is impossible to inject code or macros globally. In order to use a macro, you need to explicitly `require` or `import` the module that defines the macro.

* Macros are explicit: it is impossible to run a macro without explicitly invoking it. For example, some languages allow developers to completely rewrite functions behind the scenes, often via parse transforms or via some reflection mechanisms. In Elixir, a macro must be explicitly invoked in the caller during compilation time.

* Macros’ language is clear: many languages provide syntax shortcuts for `quote` and `unquote`. In Elixir, we preferred to have them explicitly spelled out, in order to clearly delimit the boundaries of a macro definition and its quoted expressions.

Even with such guarantees, the developer plays a big role when writing macros responsibly. If you are confident you need to resort to macros, remember that macros are not your API. Keep your macro definitions short, including their quoted contents. For example, instead of writing a macro like this:

```elixir
defmodule MyModule do
  defmacro my_macro(a, b, c) do
    quote do
      do_this(unquote(a))
      ...
      do_that(unquote(b))
      ...
      and_that(unquote(c))
    end
  end
end
```

write:

```elixir
defmodule MyModule do
  defmacro my_macro(a, b, c) do
    quote do
      # Keep what you need to do here to a minimum
      # and move everything else to a function
      do_this_that_and_that(unquote(a), unquote(b), unquote(c))
    end
  end

  def do_this_that_and_that(a, b, c) do
    do_this(a)
    ...
    do_that(b)
    ...
    and_that(c)
  end
end
```

This makes your code clearer and easier to test and maintain, as you can invoke and test `do_this_that_and_that/3` directly. It also helps you design an actual API for developers that do not want to rely on macros.

With those lessons, we finish our introduction to macros. The next chapter is a brief discussion on DSLs that shows how we can mix macros and module attributes to annotate and extend modules and functions.