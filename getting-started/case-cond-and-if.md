# case, cond и if

В этой главе мы познакомимся с управляющими конструкциями `case`, `cond` и `if`.

## `case`

`case` позволяет нам сравнивать значение с несколькими шаблонами, пока мы не найдём подходящий:

```iex
iex> case {1, 2, 3} do
...>   {4, 5, 6} ->
...>     "This clause won't match"
...>   {1, x, 3} ->
...>     "This clause will match and bind x to 2 in this clause"
...>   _ ->
...>     "This clause would match any value"
...> end
"This clause will match and bind x to 2 in this clause"
```

Если вы хотитие сопоставить значение с уже существующими переменнами, нужно использовать оператор `^`:

```iex
iex> x = 1
1
iex> case 10 do
...>   ^x -> "Won't match"
...>   _ -> "Will match"
...> end
"Will match"
```

На различные варианты также могут быть наложены дополнительные ограничивающий условия:

```iex
iex> case {1, 2, 3} do
...>   {1, x, 3} when x > 0 ->
...>     "Will match"
...>   _ ->
...>     "Would match, if guard condition were not satisfied"
...> end
"Will match"
```

Первый вариант в примере выше будет подходить только когда значение `x` положительное.

Помните, что ошибки, возникшие в ограничивающих условиях, не выбрасываются, но сразу делают невозможным соответствие шаблона, к которому они относятся:

```iex
iex> hd(1)
** (ArgumentError) argument error
iex> case 1 do
...>   x when hd(x) -> "Won't match"
...>   x -> "Got #{x}"
...> end
"Got 1"
```

Если ни один из вариантов не подходит, будет выброшена ошибка:

```iex
iex> case :ok do
...>   :error -> "Won't match"
...> end
** (CaseClauseError) no case clause matching: :ok
```

Ознакомьтесь с [документацие по ограничивающим условиям](/docs/master/elixir/guards.html), чтобы получить по ним больше информации, узнать, как их использовать, и какие выражения доступны в них.

Помните, что анонимные функции также могут иметь несколько выражений и ограничений для них:

```iex
iex> f = fn
...>   x, y when x > 0 -> x + y
...>   x, y -> x * y
...> end
#Function<12.71889879/2 in :erl_eval.expr/5>
iex> f.(1, 3)
4
iex> f.(-1, 3)
-3
```

Колечество аргументов в каждом выражении анонимной функции должно быть одинаковым, иначе будет выброшена ошибка.

```iex
iex> f2 = fn
...>   x, y when x > 0 -> x + y
...>   x, y, z -> x * y + z
...> end
** (CompileError) iex:1: cannot mix clauses with different arities in function definition
```

## `cond`

`case` полезен, когда вам нужно сопоставить различные значения. Однако, часто нам нужно проверить разные условия и найти первое, которое окажется правдой. В таком случае, можно использовать `cond`:

```iex
iex> cond do
...>   2 + 2 == 5 ->
...>     "This will not be true"
...>   2 * 2 == 3 ->
...>     "Nor this"
...>   1 + 1 == 2 ->
...>     "But this will"
...> end
"But this will"
```

Это эквивалент условию `else if` во многих императивных языках (хотя тут используется не так часто).

Если ни одно из условий не вернуло true, будет возникнет ошибка (`CondClauseError`). По этой причине может быть нужно добавить последнее условие, равное `true`, которое всегда будет подходить:

```iex
iex> cond do
...>   2 + 2 == 5 ->
...>     "This is never true"
...>   2 * 2 == 3 ->
...>     "Nor this"
...>   true ->
...>     "This is always true (equivalent to else)"
...> end
"This is always true (equivalent to else)"
```

Наконец, помните, что `cond` расценивает любые значения кроме `nil` и `false` как true:

```iex
iex> cond do
...>   hd([1, 2, 3]) ->
...>     "1 is considered as true"
...> end
"1 is considered as true"
```

## `if` и `unless`

Кроме `case` и `cond`, Elixir также предоставляет макросы `if/2` и `unless/2`, которые полезны, когда вам нужно проверить только одно условие:

```iex
iex> if true do
...>   "This works!"
...> end
"This works!"
iex> unless true do
...>   "This will never be seen"
...> end
nil
```

Если условие, переданное в `if/2`, возвращает `false` или `nil`, код между `do/end` не выполняется и возвращает `nil`. С точностью наоборот работает `unless/2`.

Они также поддерживают блоки `else`:

```iex
iex> if nil do
...>   "This won't be seen"
...> else
...>   "This will"
...> end
"This will"
```

> Обратите внимание: интересная особенность `if/2` и `unless/2` состоит в том, что они реализованы как макросы. Это не специальные языковые конструкцию, какими они являются во многих языках. Вы можете посмотреть документацию и исходный код `if/2` в [документации модуля `Kernel`](https://hexdocs.pm/elixir/Kernel.html). Модуль `Kernel` также содержит определения операторов вроде `+/2` и функций типа `is_function/2`. Они автоматически импортируются и становятся доступны в вашем коде по умолчанию.

## Блоки `do/end`

До этого момента мы изучили четыре управляющих конструкции: `case`, `cond`, `if` и `unless`, и они все обёрнуты в блоки `do/end`. Мы также могли бы написать `if` вот так:

```iex
iex> if true, do: 1 + 2
3
```

Обратите внимание, что в примере выше есть запятая между `true` и `do:`, это связано с тем, что здесь используется обычный синтаксис Elixir, в котором все аргументы перечислены через запятую. Можно сказать, что синтаксис использует *списки ключевых слов* (keyword lists). Мы можем также передать `else`: 

```iex
iex> if false, do: :this, else: :that
:that
```

Блоки `do/end` - синтетическое удобство, построенное над ключевыми словами (syntactic convenience built on top of the keywords one). Поэтому блоки `do/end` не нуждаются в запятой между предыдущим аргументом и блоком. Они полезны, т.к. делают более чистой запись блоков кода. Следующие примеры эквивалентны:

```iex
iex> if true do
...>   a = 1 + 2
...>   a + 10
...> end
13
iex> if true, do: (
...>   a = 1 + 2
...>   a + 10
...> )
13
```

Одна вещь, которую следует помнить при использовании блоков `do/end`: они всегда связываются с самым удалённым вызовом функции. Например, следующее выражение:

```iex
iex> is_number if true do
...>  1 + 2
...> end
** (CompileError) undefined function: is_number/2
```

Было бы распарсено как:

```iex
iex> is_number(if true) do
...>  1 + 2
...> end
** (CompileError) undefined function: is_number/2
```

which leads to an undefined function error because that invocation passes two arguments, and `is_number/2` does not exist. The `if true` expression is invalid in itself because it needs the block, but since the arity of `is_number/2` does not match, Elixir does not even reach its evaluation.

что привео бы к ошибке, т.к. функции `is_number/2` не существует. Выражение `if true` не корректно само по себе, ему необходим блок, но ввиду того, что арность `is_number/2` не соответствует арности существующей функции, Elixir просто не дойдёт до этого выражения.

Добавим недостающие скобки, чтобы привязать блок к `if`:

```iex
iex> is_number(if true do
...>  1 + 2
...> end)
true
```

Списки ключевых слов играют важную роль в языке и часто используются во многих функция и макросах. Мы ознакомимся с ними подробнее в с будущих главах. Теперь самое время поговорить о "Двоичных данных, строках и списках символов".