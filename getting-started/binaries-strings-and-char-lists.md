# Двоичные данные, строки и списки символов

В "Базовых типа" мы познакомились со строками и использовали функцию `is_binary/1` для проверок:

```iex
iex> string = "hello"
"hello"
iex> is_binary(string)
true
```

В этой главе мы поймём, что такое двоичные данные (binaries), как они связаны со строками, и что такое значения в одинарных кавычках `'like this'` в Elixir.

## UTF-8 и Юникод

Строка в UTF-8 представляет собой бинарную последовательность. Для понимания, что мы подразумеваем, следует понять разницу между байтами и code points (кодовые обозначения?).

Стандарт Unicode связывает кодовые обозначения со многими известными символами. Например, латинская буква `a` имеет кодовое обозначение `97`, тогда как буква `ł` имеет код `322`. Чтобы записать строку `"hełło"` на диск, нужно конвертировать эти коды в байты. Если мы примем за правило, что один байт соответствует одному кодовому обозначению, мы не сможем записать строку `"hełło"`, потому что она использует код `322` для `ł`, а один байт может представлять число от `0` до `255`. Но, раз мы можем прочитать `"hełło"` на экране, есть *какой-то способ* записать такую строку. Это место, где начинает работать кодировка.

Для представления кодовых обозначений в виде байт, нужно как-то их закодировать. Elixir выбрал кодировку UTF-8 своей главной и стандартной кодировкой. Когда мы говорим, что UTF-8 закодирован в двоичном виде, мы имеем ввиду, что строка - это набор байт, представляющих некоторые кодовые обозначения, в соответствии с правилами кодировки UTF-8.

Т.к. у нас есть символы вроде `ł`, соответствующий коду `322`, нам нужно больше одного байта для их представления. Поэтому мы видим разницу при вычислении `byte_size/1` в сравнении с результатом `String.length/1`:

```iex
iex> string = "hełło"
"hełło"
iex> byte_size(string)
7
iex> String.length(string)
5
```

Так, `byte_size/1` считает количество байт, необходимых для представления строки, а `String.length/1` считает символы.

> Обратите внимание: если вы работаете под Windows, есть шанс, что ваш терминал не использует UTF-8 по умолчанию. Вы можете изменить кодировку текущей сессии, выполнив `chcp 65001` перед запуском `iex` (`iex.bat`).

UTF-8 нужен один байт для представления символов `h`, `e`, и `o`, но два байта для представления `ł`. В Elixir вы можете узнать код символа с помощью `?`:

```iex
iex> ?a
97
iex> ?ł
322
```

Вы можете также использовать функции из [модуля `string`](https://hexdocs.pm/elixir/String.html) для разделения строки на индивидуальные символы, каждый из которых будет строкой длины 1:

```iex
iex> String.codepoints("hełło")
["h", "e", "ł", "ł", "o"]
```

Вы увидете, что Elixir имеет отличную поддержку работы со строками. Он также поддерживает многие операции Юникода. Фактически, Elixir успешно проходит все тесты, показанные в статье ["The string type is broken"](http://mortoray.com/2013/11/27/the-string-type-is-broken/).

Однако, строки - это только часть истории. Раз строки являются бинарными, и мы использовали функцию `is_binary/1`, Elixir должен иметь более мощный тип, лежащий в основе строк. И он есть! Поговорим о бинарных данных.

## Binaries (and bitstrings)

In Elixir, you can define a binary using `<<>>`:

```iex
iex> <<0, 1, 2, 3>>
<<0, 1, 2, 3>>
iex> byte_size(<<0, 1, 2, 3>>)
4
```

A binary is a sequence of bytes. Those bytes can be organized in any way, even in a sequence that does not make them a valid string:

```iex
iex> String.valid?(<<239, 191, 191>>)
false
```

The string concatenation operation is actually a binary concatenation operator:

```iex
iex> <<0, 1>> <> <<2, 3>>
<<0, 1, 2, 3>>
```

A common trick in Elixir is to concatenate the null byte `<<0>>` to a string to see its inner binary representation:

```iex
iex> "hełło" <> <<0>>
<<104, 101, 197, 130, 197, 130, 111, 0>>
```

Each number given to a binary is meant to represent a byte and therefore must go up to 255. Binaries allow modifiers to be given to store numbers bigger than 255 or to convert a code point to its UTF-8 representation:

```iex
iex> <<255>>
<<255>>
iex> <<256>> # truncated
<<0>>
iex> <<256 :: size(16)>> # use 16 bits (2 bytes) to store the number
<<1, 0>>
iex> <<256 :: utf8>> # the number is a code point
"Ā"
iex> <<256 :: utf8, 0>>
<<196, 128, 0>>
```

If a byte has 8 bits, what happens if we pass a size of 1 bit?

```iex
iex> <<1 :: size(1)>>
<<1::size(1)>>
iex> <<2 :: size(1)>> # truncated
<<0::size(1)>>
iex> is_binary(<<1 :: size(1)>>)
false
iex> is_bitstring(<<1 :: size(1)>>)
true
iex> bit_size(<< 1 :: size(1)>>)
1
```

The value is no longer a binary, but a bitstring -- a bunch of bits! So a binary is a bitstring where the number of bits is divisible by 8.

```iex
iex>  is_binary(<<1 :: size(16)>>)
true
iex>  is_binary(<<1 :: size(15)>>)
false
```

We can also pattern match on binaries / bitstrings:

```iex
iex> <<0, 1, x>> = <<0, 1, 2>>
<<0, 1, 2>>
iex> x
2
iex> <<0, 1, x>> = <<0, 1, 2, 3>>
** (MatchError) no match of right hand side value: <<0, 1, 2, 3>>
```

Note each entry in the binary pattern is expected to match exactly 8 bits. If we want to match on a binary of unknown size, it is possible by using the binary modifier at the end of the pattern:

```iex
iex> <<0, 1, x :: binary>> = <<0, 1, 2, 3>>
<<0, 1, 2, 3>>
iex> x
<<2, 3>>
```

Similar results can be achieved with the string concatenation operator `<>`:

```iex
iex> "he" <> rest = "hello"
"hello"
iex> rest
"llo"
```

A complete reference about the binary / bitstring constructor `<<>>` can be found [in the Elixir documentation](https://hexdocs.pm/elixir/Kernel.SpecialForms.html#%3C%3C%3E%3E/1). This concludes our tour of bitstrings, binaries and strings. A string is a UTF-8 encoded binary and a binary is a bitstring where the number of bits is divisible by 8. Although this shows the flexibility Elixir provides for working with bits and bytes, 99% of the time you will be working with binaries and using the `is_binary/1` and `byte_size/1` functions.

## Char lists

A char list is nothing more than a list of code points. Char lists may be created with single-quoted literals:

```iex
iex> 'hełło'
[104, 101, 322, 322, 111]
iex> is_list 'hełło'
true
iex> 'hello'
'hello'
iex> List.first('hello')
104
```

You can see that, instead of containing bytes, a char list contains the code points of the characters between single-quotes (note that by default IEx will only output code points if any of the integers is outside the ASCII range). So while double-quotes represent a string (i.e. a binary), single-quotes represent a char list (i.e. a list).

In practice, char lists are used mostly when interfacing with Erlang, in particular old libraries that do not accept binaries as arguments. You can convert a char list to a string and back by using the `to_string/1` and `to_charlist/1` functions:

```iex
iex> to_charlist "hełło"
[104, 101, 322, 322, 111]
iex> to_string 'hełło'
"hełło"
iex> to_string :hello
"hello"
iex> to_string 1
"1"
```

Note that those functions are polymorphic. They not only convert char lists to strings, but also integers to strings, atoms to strings, and so on.

With binaries, strings, and char lists out of the way, it is time to talk about key-value data structures.