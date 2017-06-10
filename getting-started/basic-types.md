# Базовые типы

В этой главе мы узнаем больше о базовых типах Elixir: целых числах (integers), числах с плавающей запятой (floats), логических или булевых значениях (booleans), атомах (atoms), строках (strings), списках (lists) и кортежах (tuples). Вот некоторые базовые типы:

```iex
iex> 1          # integer
iex> 0x1F       # integer
iex> 1.0        # float
iex> true       # boolean
iex> :atom      # atom / symbol
iex> "elixir"   # string
iex> [1, 2, 3]  # list
iex> {1, 2, 3}  # tuple
```

## Базовая арифметика

Откройте `iex` и введите следующие выражения:

```iex
iex> 1 + 2
3
iex> 5 * 5
25
iex> 10 / 2
5.0
```

Обратите внимание, что `10 / 2` вернёт число с плавающей запятой `5.0`, а не целое `5`. Это ожидаемо. В Elixir оператор `/` всегда возвращает float. Если вы хотите совершить целочисленное деление или получить остаток от деления, вы можете использовать функции `div` и `rem` (от англ. division и remainder):

```iex
iex> div(10, 2)
5
iex> div 10, 2
5
iex> rem 10, 3
1
```

Elixir позволяет опустить скобки при вызове именованных функций. Такая возможность позволяет получить более читаемый синтаксис при объявлении синтакса и написаниее control-flow constructions (?).

Elixir также поддерживает краткую форму написания двоичных, восьмиричных и шестнадцатеричных чисел:

```iex
iex> 0b1010
10
iex> 0o777
511
iex> 0x1F
31
```

Числа с плавающей запятой обязательно должны содержать точку и хотя бы одну цифру после неё, а также поддерживают `e` для экспоненциальных чисел:

```iex
iex> 1.0
1.0
iex> 1.0e-10
1.0e-10
```

Числа с плавающей запятов в Elixir занимают 64 бита и имеют двойную точность.

Вы можете выполнить функцию `round`, чтобы получить ближайшее целое число к аргументу с типом float, или функцию `trunc`, которая вернёт целую часть от числа.

```iex
iex> round(3.58)
4
iex> trunc(3.58)
3
```

## Идентификация функций

Функции в Elixir идентифицируются по их именам и их арности (arity). Арность функции - это количество аргументов, которые принимает функция. С этого момента мы будем использовать как имя функции, так и её арность для обозначения функции во всей документации. `round/1` обозначает функцию с именем `round`, которая принимает 1 аргумент, в то время как `round/2` обозначает другую (несуществующую) функцию с тем же именем, но с арностью `2`.

## Логические (булевы) значения

Elixir поддерживает `true` и `false` в качестве булевых значений:

```iex
iex> true
true
iex> true == false
false
```

Elixir предоставляет набор функций для проверки типа. Например, функция `is_boolean/1` может быть использована для проверки, является значение булевым или нет:

```iex
iex> is_boolean(true)
true
iex> is_boolean(1)
false
```

Вы можете также использовать `is_integer/1`, `is_float/1` или `is_number/1`, чтобы проверить, является ли аргумент целым числом, дробным числом или одним из них.

> Помните: в любой момент вы можете набрать `h()` в интерактивной оболочке и получить информацию информацию, как её использовать. Хелпер `h` может также быть использован для доступа к документации по любой функции. Например, команда `h is_integer/1` напечатает документация функции `is_integer/1`. Это также работает для операторов и других конструкций (попробуйте `h ==/2`).

## Атомы

Атомы - это константы, имя которых является также и их значением. Некоторые другие языки называют их символами (symbols):

```iex
iex> :hello
:hello
iex> :hello == :world
false
```

Булевы значения `true` и `false`, фактически, являются атомами:

```iex
iex> true == :true
true
iex> is_atom(false)
true
iex> is_boolean(:false)
true
```

## Строки

Строки в Elixir располагаются внутри двойных кавычек, и они представлены в кодировке UTF-8:

```iex
iex> "hellö"
"hellö"
```

> Обратите внимание: если вы работаете под Windows, ваш терминал может использовать отличную от UTF-8 кодировку по умолчанию. Вы можете сменить кодировку текущей сессии командой `chcp 65001` перед запуском IEx.

Elixir также поддерживает интерполяцию строк:

```iex
iex> "hellö #{:world}"
"hellö world"
```

Строки могут содержать разрывы. Вы можете ввести их, используя escape sequences:

```iex
iex> "hello
...> world"
"hello\nworld"
iex> "hello\nworld"
"hello\nworld"
```

Вы можете вывести строку, используя функцию `IO.puts/1` из модуля `IO`:

```iex
iex> IO.puts "hello\nworld"
hello
world
:ok
```

Помните, что функция `IO.puts/1` возвращает атом `:ok` в качестве результата после вывода.

Строки в Elixir представлены внутри двоичными данными (binaries), которые являются последовательностями байтов:

```iex
iex> is_binary("hellö")
true
```

Мы также можем получить количество байт в строке:

```iex
iex> byte_size("hellö")
6
```

Обратите внимание, что количество байт в данной строке равно 6, хотя в ней 5 символов. Дело в том, что символ "ö" занимает 2 байта, чтобы быть представленным в UTF-8. Мы можем получить реальную длину строки, основанную на количестве символов, использовав функцию `String.length/1`:

```iex
iex> String.length("hellö")
5
```

[Модуль String](https://hexdocs.pm/elixir/String.html) содержит набор функций для манипуляций со строками, удовлетворяющий стандарту Unicode:

```iex
iex> String.upcase("hellö")
"HELLÖ"
```

## Анонимные функции

Анонимные функции могут быть созданы в одну строку, они заключаются между ключевых слов `fn` и `end`:

```iex
iex> add = fn a, b -> a + b end
#Function<12.71889879/2 in :erl_eval.expr/5>
iex> add.(1, 2)
3
iex> is_function(add)
true
iex> is_function(add, 2) # check if add is a function that expects exactly 2 arguments
true
iex> is_function(add, 1) # check if add is a function that expects exactly 1 argument
false
```

Функции в Elixir являются объектами первого класса, то есть они могут быть переданы в качестве аргумента другой функции, так же, как целые числа и строки. В примере мы передали функции `is_function/1` переменную `add` и она вернула `true`. Мы можем также проверить арность функции вызовом `is_function/2`.

Note a dot (`.`) between the variable and parentheses is required to invoke an anonymous function. The dot ensures there is no ambiguity between calling an anonymous function named `add` and a named function `add/2`. In this sense, Elixir makes a clear distinction between anonymous functions and named functions. We will explore those differences in [Chapter 8](/getting-started/modules-and-functions.html).

Обратите внимание, что точка (`.`) между переменной и скобками обязательна для вызова анонимной функции. Эта точка нужна, чтобы избавиться от неоднозначности между анонимной функцией `add` и именованной `add/2`. В этом смысле в Elixir явно различаются анонимные и именованные функции. Мы поговорим об этой разнице в [Главе 8](/getting-started/modules-and-functions.html).

Анонимные функции являются замыканиями и по существу они могут иметь доступ к переменным из области видимости (scope), когда функция объявлена. Давайте объявим новую анонимную функцию, которая использует анонимную функцию `add`, которую мы объявили ранее:

```iex
iex> double = fn a -> add.(a, a) end
#Function<6.71889879/1 in :erl_eval.expr/5>
iex> double.(2)
4
```

Keep in mind a variable assigned inside a function does not affect its surrounding environment:

Помните, что присвоения значений внутри функции никак не изменяют её окружение:

```iex
iex> x = 42
42
iex> (fn -> x = 0 end).()
0
iex> x
42
```

## (Связные) Списки

Elixir использует квадратные скобки, чтобы задать список значений. Значения могут быть любого типа:

```iex
iex> [1, 2, true, 3]
[1, 2, true, 3]
iex> length [1, 2, 3]
3
```

Два списка могут быть сложены, используя оператор `++/2`, и один может быть вычтен из другого с помощью `--/2`:

```iex
iex> [1, 2, 3] ++ [4, 5, 6]
[1, 2, 3, 4, 5, 6]
iex> [1, true, 2, false, 3, true] -- [true, false]
[1, 2, 3, true]
```

В этом руководстве мы будем много говорить о голове и хвосте списка. Головой называют первый элемент списка, а хвостом - его оставшуюся часть. Они могут быть получены функциями `hd/1` и `tl/1`. Давайте создадим список и получим его голову и хвост:

```iex
iex> list = [1, 2, 3]
iex> hd(list)
1
iex> tl(list)
[2, 3]
```

Попытка получить голову или хвост пустого списка выбросит ошибку:

```iex
iex> hd []
** (ArgumentError) argument error
```

Иногда вы будете создавать список и он будет возвращать значение в одинарных кавычках. Например:

```iex
iex> [11, 12, 13]
'\v\f\r'
iex> [104, 101, 108, 108, 111]
'hello'
```

Когда Elixir видит список корректных ASCII кодов, которые может напечатать, он выводит список символов (в буквальном смысле). Списки символов часто используются для взаимодействия с существующим кодом на Erlang. Если вы видете значение в IEx, и вы не уверены, что это такое, вы можете использовать `i/1` для получения информации:

```iex
iex> i 'hello'
Term
  'hello'
Data type
  List
Description
  ...
Raw representation
  [104, 101, 108, 108, 111]
Reference modules
  List
```

Помните, что значения в одинарных и двойных кавычках не эквивалентны в Elixir, они принадлежат разным типам:

```iex
iex> 'hello' == "hello"
false
```

Внутри одиночных кавычек списки символов (char lists), внутри двойных - строки. Мы поговорим обы этом больше в главе ["Бинарные данные, строки и списки символов"](/getting-started/binaries-strings-and-char-lists.html)

## Tuples

Elixir uses curly brackets to define tuples. Like lists, tuples can hold any value:

```iex
iex> {:ok, "hello"}
{:ok, "hello"}
iex> tuple_size {:ok, "hello"}
2
```

Tuples store elements contiguously in memory. This means accessing a tuple element by index or getting the tuple size is a fast operation. Indexes start from zero:

```iex
iex> tuple = {:ok, "hello"}
{:ok, "hello"}
iex> elem(tuple, 1)
"hello"
iex> tuple_size(tuple)
2
```

It is also possible to put an element at a particular index in a tuple with `put_elem/3`:

```iex
iex> tuple = {:ok, "hello"}
{:ok, "hello"}
iex> put_elem(tuple, 1, "world")
{:ok, "world"}
iex> tuple
{:ok, "hello"}
```

Notice that `put_elem/3` returned a new tuple. The original tuple stored in the `tuple` variable was not modified because Elixir data types are immutable. By being immutable, Elixir code is easier to reason about as you never need to worry that any code might be mutating your data structure in place.

## Lists or tuples?

What is the difference between lists and tuples?

Lists are stored in memory as linked lists, meaning that each element in a list holds its value and points to the following element until the end of the list is reached. We call each pair of value and pointer a **cons cell**:

```iex
iex> list = [1 | [2 | [3 | []]]]
[1, 2, 3]
```

This means accessing the length of a list is a linear operation: we need to traverse the whole list in order to figure out its size. Updating a list is fast as long as we are prepending elements:

```iex
iex> [0 | list]
[0, 1, 2, 3]
```

Tuples, on the other hand, are stored contiguously in memory. This means getting the tuple size or accessing an element by index is fast. However, updating or adding elements to tuples is expensive because it requires copying the whole tuple in memory.

Those performance characteristics dictate the usage of those data structures. One very common use case for tuples is to use them to return extra information from a function. For example, `File.read/1` is a function that can be used to read file contents. It returns tuples:

```iex
iex> File.read("path/to/existing/file")
{:ok, "... contents ..."}
iex> File.read("path/to/unknown/file")
{:error, :enoent}
```

If the path given to `File.read/1` exists, it returns a tuple with the atom `:ok` as the first element and the file contents as the second. Otherwise, it returns a tuple with `:error` and the error description.

Most of the time, Elixir is going to guide you to do the right thing. For example, there is an `elem/2` function to access a tuple item but there is no built-in equivalent for lists:

```iex
iex> tuple = {:ok, "hello"}
{:ok, "hello"}
iex> elem(tuple, 1)
"hello"
```

When counting the elements in a data structure, Elixir also abides by a simple rule: the function is named `size` if the operation is in constant time (i.e. the value is pre-calculated) or `length` if the operation is linear (i.e. calculating the length gets slower as the input grows). As a mnemonic, both "length" and "linear" start with "l".

For example, we have used 4 counting functions so far: `byte_size/1` (for the number of bytes in a string), `tuple_size/1` (for tuple size), `length/1` (for list length) and `String.length/1` (for the number of graphemes in a string). We use `byte_size` to get the number of bytes in a string -- a cheap operation. Retrieving the number of unicode characters, on the other hand, uses `String.length`, and may be expensive as it relies on a traversal of the entire string.

Elixir also provides `Port`, `Reference`, and `PID` as data types (usually used in process communication), and we will take a quick look at them when talking about processes. For now, let's take a look at some of the basic operators that go with our basic types.
