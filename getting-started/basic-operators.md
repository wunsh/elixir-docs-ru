# Базовые операторы

В [предыдущей главе](/getting-started/basic-types.html) мы увидели, что Elixir предоставляет операторы `+`, `-`, `*`, `/` для арифметических операций, а также функции `div/2` и `rem/2` для целочисленного деления и получения остатка.

Elixir также предоставляет `++` и `--` для манипуляций со списками:

```iex
iex> [1, 2, 3] ++ [4, 5, 6]
[1, 2, 3, 4, 5, 6]
iex> [1, 2, 3] -- [2]
[1, 3]
```

Конкатенация строк выполняется с помощью `<>`:

```iex
iex> "foo" <> "bar"
"foobar"
```

Elixir also provides three boolean operators: `or`, `and` and `not`. These operators are strict in the sense that they expect a boolean (`true` or `false`) as their first argument:

Elixir также предоставляет три логических оператора: `or`, `and` и `not`. Эти операторы строгие в том смысле, что они ожидают булево значение (`true` or `false`) в качестве первого аргумента:

```iex
iex> true and true
true
iex> false or is_atom(:example)
true
```

Передача значения другого типа вызовет исключение:

```iex
iex> 1 and true
** (BadBooleanError) expected a boolean on left-side of "and", got: 1
```

`or` и `and` - операторы сокращенного вычисления. Они выполняют правую часть только если левой не достаточно, чтобы определить результат:

```iex
iex> false and raise("This error will never be raised")
false
iex> true or raise("This error will never be raised")
true
```

> Обратите внимание: если вы Erlang разработчик, `and` и `or` в Elixir на самом деле соответствуют операторам `andalso` и `orelse` в Erlang.

Кроме этих операторов, Elixir также предоставляет `||`, `&&` и `!`, которые принимают аргументы любого типа. Для этих операторов все значения, кроме `false` и `nil` будут возвращать true:

```iex
# or
iex> 1 || true
1
iex> false || 11
11

# and
iex> nil && 13
nil
iex> true && 17
17

# !
iex> !true
false
iex> !1
false
iex> !nil
true
```

Подытожим: используйте `and`, `or` и `not`, когда вы ожидаете булевы значения. Если любой из аргументов не булева типа, используйте `&&`, `||` и `!`.

Elixir также предоставляет операторы сравнения `==`, `!=`, `===`, `!==`, `<=`, `>=`, `<`, и `>`:

```iex
iex> 1 == 1
true
iex> 1 != 2
true
iex> 1 < 2
true
```

Разница между `==` и `===` состоит в том, что последний более строг при сравнении целых и дробных чисел:

```iex
iex> 1 == 1.0
true
iex> 1 === 1.0
false
```

В Elixir мы можем сравнивать два разных типа данных:

```iex
iex> 1 < :atom
true
```

Причина, по которой мы можем это сделать - прагматизм. Алгоритмам сортировки не нужно беспокоиться о разнице типов при сортировке. Полный порядок сортировки представлен ниже:

    number < atom < reference < function < port < pid < tuple < map < list < bitstring

Вам не нужно запоминать этот порядок, достаточно знать, что он существует.

For reference information about operators (and ordering), check the [reference page on operators](/docs/master/elixir/operators.html).

Для получения справочной информации по операторам (и порядку сортировки), перейдите на [страницу справки по операторам](/docs/master/elixir/operators.html).

В следующей главе, мы собираемся обсудить некоторые базовые функции, приведение типов данных и немного об управлении потоком исполнения.
