---
title: Библиотеки Эрланга
next_page: where-to-go-next
prev_page: typespecs-and-behaviours
---

Эликсир отлично поддерживает работу с библиотеками Эрланга. Фактически, Эликсир отбивает желание писать обёртки для библиотек Эрланга, сделав удобным прямое взаимодействие с Эрланг кодом. В данном разделе мы покажем наиболее полезную функциональность из Эрланга, которой нет в Эликсире.

Когда вы познакомитесь с Эликсиром достаточно хорошо, вероятно, вы захотите ознакомиться с [STDLIB Reference Manual](http://erlang.org/doc/apps/stdlib/index.html) более подробно.

## Модуль Binary

Встроенный в Эликсир модуль String работает с бинарными данными, кодирующими UTF-8. [Модуль Binary](http://erlang.org/doc/man/binary.html) полезен, когда вам нужно работать с бинарными данными, которые необязательно представляют Юникод.

```elixir
iex> String.to_charlist "Ø"
[216]
iex> :binary.bin_to_list "Ø"
[195, 152]
```

Пример выше показывает разницу; модуль `String` Возвращает значения символов Юникода, тогда как `:binary` работает с сырыми байтами данных.

## Форматированный текстовый вывод

В Эликсире нет функции вроде `printf` из C и других языков. К счастью, в стандартной библиотеке Эрланга есть функции `:io.format/2` и `:io_lib.format/2`. Первая форматирует вывод терминала, тогда как вторая работает с `iolist`. Спецификаторы форматирования отличаются от `printf`, [подробности можно узнать в документации Эрланга](http://erlang.org/doc/man/io.html#format-1).

```elixir
iex> :io.format("Pi is approximately given by:~10.3f~n", [:math.pi])
Pi is approximately given by:     3.142
:ok
iex> to_string :io_lib.format("Pi is approximately given by:~10.3f~n", [:math.pi])
"Pi is approximately given by:     3.142\n"
```

Также помните, что функции форматирования в Эрланге уделяют особое внимание обработке Юникода.

## Модуль `crypto`

[Модуль crypto](http://erlang.org/doc/man/crypto.html) содержит функции хэширования, цифровых подписей, шифрования и другие:

```elixir
iex> Base.encode16(:crypto.hash(:sha256, "Elixir"))
"3315715A7A3AD57428298676C5AE465DADA38D951BDFAC9348A8A31E9C7401CB"
```

Модуль `:crypto` не является частью стандартной библиотеки Эрланга, но содержится в стандартной поставке. Это значит, что вам нужно указать `:crypto` в вашем приложении, когда вы хотите его использовать. Чтобы это сделать, отредактируйте файл `mix.exs`, чтобы он включал следующее:

```elixir
def application do
  [extra_applications: [:crypto]]
end
```

## Модуль `digraph`

[Модуль digraph](http://erlang.org/doc/man/digraph.html) (также как и [digraph_utils](http://erlang.org/doc/man/digraph_utils.html)) содержит функции для работы с направленными графами, состоящими из вершин и рёбер. После построения графа, алгоритмы в нём помогут найти, например, кратчайший путь между двумя вершинами или зацикленность в графе.

Дано три вершины, найдём кратчайший путь из первой к последней.

```elixir
iex> digraph = :digraph.new()
iex> coords = [{0.0, 0.0}, {1.0, 0.0}, {1.0, 1.0}]
iex> [v0, v1, v2] = (for c <- coords, do: :digraph.add_vertex(digraph, c))
iex> :digraph.add_edge(digraph, v0, v1)
iex> :digraph.add_edge(digraph, v1, v2)
iex> :digraph.get_short_path(digraph, v0, v2)
[{0.0, 0.0}, {1.0, 0.0}, {1.0, 1.0}]
```

Обратите внимание, что функции в `:digraph` изменяют структуру графа налету, это возможно благодаря использованию таблиц ETS, которые мы рассмотрим следующим пунктом.

## Erlang Term Storage

Модули [`ets`](http://erlang.org/doc/man/ets.html) и [`dets`](http://erlang.org/doc/man/dets.html) позволяют хранить большие структуры данных в памяти или на диске соответственно.

ETS позволяет вам создать таблицу, содержащую кортежи. По умолчанию, таблицы ETS защищены, это значит, что только владелец процесса может писать в таблицу, но любой другой процесс может её читать. В ETS есть функциональность, позволяющая использовать их как простую базу данных, хранилище пар ключ-значение или в качестве механизма кеширования.

Функции в модуле `ets` будут менять состояние таблице как сайд-эффект.

```elixir
iex> table = :ets.new(:ets_test, [])
# Store as tuples with {name, population}
iex> :ets.insert(table, {"China", 1_374_000_000})
iex> :ets.insert(table, {"India", 1_284_000_000})
iex> :ets.insert(table, {"USA", 322_000_000})
iex> :ets.i(table)
<1   > {<<"India">>,1284000000}
<2   > {<<"USA">>,322000000}
<3   > {<<"China">>,1374000000}
```

## Модуль `math`

[Модуль `math`](http://erlang.org/doc/man/math.html) содержит основные математические операции: тригонометрия, экспоненциальные и логарифмические функции.

```elixir
iex> angle_45_deg = :math.pi() * 45.0 / 180.0
iex> :math.sin(angle_45_deg)
0.7071067811865475
iex> :math.exp(55.0)
7.694785265142018e23
iex> :math.log(7.694785265142018e23)
55.0
```

## Модуль `queue`

[Очередь или `queue`](http://erlang.org/doc/man/queue.html) - структура данных, которая эффективно реализует очереди FIFO (first-in first-out):

```elixir
iex> q = :queue.new
iex> q = :queue.in("A", q)
iex> q = :queue.in("B", q)
iex> {value, q} = :queue.out(q)
iex> value
{:value, "A"}
iex> {value, q} = :queue.out(q)
iex> value
{:value, "B"}
iex> {value, q} = :queue.out(q)
iex> value
:empty
```

## Модуль `rand`

[В `rand` есть функции](http://erlang.org/doc/man/rand.html) для возврата случайного значения или получения произвольного элемента множества.

```elixir
iex> :rand.uniform()
0.8175669086010815
iex> _ = :rand.seed(:exs1024, {123, 123534, 345345})
iex> :rand.uniform()
0.5820506340260994
iex> :rand.uniform(6)
6
```

## Модули `zip` и `zlib`

[Модуль `zip`](http://erlang.org/doc/man/zip.html) позволяет вам читать и записывать ZIP файлы на диск или в память, а также извлекать информацию из этих файлов

Этот код подсчитывает количество файлов в ZIP файле:

```elixir
iex> :zip.foldl(fn _, _, _, acc -> acc + 1 end, 0, :binary.bin_to_list("file.zip"))
{:ok, 633}
```

[Модуль `zlib`](http://erlang.org/doc/man/zlib.html) работает со сжатием данных в формате zlib, тем же, с которым работает команда `gzip`.

```elixir
iex> song = "
...> Mary had a little lamb,
...> His fleece was white as snow,
...> And everywhere that Mary went,
...> The lamb was sure to go."
iex> compressed = :zlib.compress(song)
iex> byte_size song
110
iex> byte_size compressed
99
iex> :zlib.uncompress(compressed)
"\nMary had a little lamb,\nHis fleece was white as snow,\nAnd everywhere that Mary went,\nThe lamb was sure to go."
```
