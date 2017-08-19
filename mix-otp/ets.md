---
title: ETS
---

# {{ page.title }}

Каждый раз, когда нам нужно обратиться к корзине, мы вынждены отправлять сообщение в реестр. В случае, если к реестру конкуретно обращаются несколько процессов, он рискует стать узким местом!

В этой главе мы поговорим о ETS (Erlang Term Storage, хранилище термов Эрланга), и о том, как его использовать для кеширования

> Осторожно! Не начинайте использовать ETS для кеша преждевременно! Логируйте и анализируйте производительность вашего приложения и ищите узкие места, чтобы знать, *нужен ли* вам кеш, и *что* вам нужно кешировать. Эта глава - просто пример, как можно использовать ETS, когда вы поймёте, что это вам действительно нужно.

## ETS для кеша

ETS allows us to store any Elixir term in an in-memory table. Working with ETS tables is done via [Erlang's `:ets` module](http://www.erlang.org/doc/man/ets.html):

ETS позволяет нам хранить любой терм Эликсира в таблице в памяти. Работа с таблицами ETS возможна через [модуль Эрланга `:ets`](http://www.erlang.org/doc/man/ets.html):

```iex
iex> table = :ets.new(:buckets_registry, [:set, :protected])
8207
iex> :ets.insert(table, {"foo", self()})
true
iex> :ets.lookup(table, "foo")
[{"foo", #PID<0.41.0>}]
```

Для создания таблицы ETS обязательны два аргумента: имя таблицы и набор опций. Из доступных опций мы передали тип таблицы и правила доступа к ней. Мы выбрали тип `:set`, который запрещает дублирование ключей. Мы также обозначили доступ к таблице как `:protected`, чтобы позволить записывать данные в таблицу только процессу, который её создал, а все остальные могут только читать эти данные. Эти значения стандартные, поэтому далее мы их будем опускать.

Таблицы ETS могут быть также именованными, позволяя получать доступ к ним, передавая имя:

```iex
iex> :ets.new(:buckets_registry, [:named_table])
:buckets_registry
iex> :ets.insert(:buckets_registry, {"foo", self()})
true
iex> :ets.lookup(:buckets_registry, "foo")
[{"foo", #PID<0.41.0>}]
```

Давайте изменим `KV.Registry`, использовав таблицы ETS. Первое изменение - обязательный аргумент `name`, который мы будем использовать для именования таблицы ETS и самого процесса реестра. Имена процессов и таблиц ETS хранятся в разных местах, поэтому конфликты между ними не возникнут.

Откройте `lib/kv/registry.ex` и измените его реализацию. Мы добавили комметарии в исходный код, чтобы увидеть все сделанные изменения:

```elixir
defmodule KV.Registry do
  use GenServer

  ## Client API

  @doc """
  Starts the registry with the given options.

  `:name` is always required.
  """
  def start_link(opts) do
    # 1. Pass the name to GenServer's init
    server = Keyword.fetch!(opts, :name)
    GenServer.start_link(__MODULE__, server, opts)
  end

  @doc """
  Looks up the bucket pid for `name` stored in `server`.

  Returns `{:ok, pid}` if the bucket exists, `:error` otherwise.
  """
  def lookup(server, name) do
    # 2. Lookup is now done directly in ETS, without accessing the server
    case :ets.lookup(server, name) do
      [{^name, pid}] -> {:ok, pid}
      [] -> :error
    end
  end

  @doc """
  Ensures there is a bucket associated to the given `name` in `server`.
  """
  def create(server, name) do
    GenServer.cast(server, {:create, name})
  end

  ## Server callbacks

  def init(table) do
    # 3. We have replaced the names map by the ETS table
    names = :ets.new(table, [:named_table, read_concurrency: true])
    refs  = %{}
    {:ok, {names, refs}}
  end

  # 4. The previous handle_call callback for lookup was removed

  def handle_cast({:create, name}, {names, refs}) do
    # 5. Read and write to the ETS table instead of the map
    case lookup(names, name) do
      {:ok, _pid} ->
        {:noreply, {names, refs}}
      :error ->
        {:ok, pid} = KV.BucketSupervisor.start_bucket()
        ref = Process.monitor(pid)
        refs = Map.put(refs, ref, name)
        :ets.insert(names, {name, pid})
        {:noreply, {names, refs}}
    end
  end

  def handle_info({:DOWN, ref, :process, _pid, _reason}, {names, refs}) do
    # 6. Delete from the ETS table instead of the map
    {name, refs} = Map.pop(refs, ref)
    :ets.delete(names, name)
    {:noreply, {names, refs}}
  end

  def handle_info(_msg, state) do
    {:noreply, state}
  end
end
```

Обратите внимание, что до изменений `KV.Registry.lookup/2` отправлял запросы на сервер, но теперь он читает прямо из таблицы ETS, которая доступна всем процессам. Это главная идея механизма кеширования, который мы делаем.

Для работы механизма кеширования созданная таблица ETS должна иметь политику доступа `:protected` (по умолчанию), чтобы все клиенты могли читать из неё данные, тогда как только процесс `KV.Registry` записывает их. Мы также можем установить `read_concurrency: true` при создании таблицы, оптимизировав таблицу для частых сценариев параллельных операций чтения.

Сделанные нами изменения сломали тесты, потому что реестр теперь должен принимать опцию `name:` при старте. Более того, некоторые операции с реестром, такие как `lookup/2` тоже должны принимать `name` в качестве аргумента, вместо PID, т.к. поиск теперь нужно делать в таблице ETS. Давайте изменим эти функции в `test/kv/registry_test.exs`, чтобы решить обе проблемы:

```elixir
  setup context do
    {:ok, _} = start_supervised({KV.Registry, name: context.test})
    %{registry: context.test}
  end
```

После изменения блока `setup` некоторые тесты продолжат проваливаться. Вы можете отметить, что тесты проходят и валятся не одинаково от запуска к запуску. Например, тест "spawns buckets":

```elixir
test "spawns buckets", %{registry: registry} do
  assert KV.Registry.lookup(registry, "shopping") == :error

  KV.Registry.create(registry, "shopping")
  assert {:ok, bucket} = KV.Registry.lookup(registry, "shopping")

  KV.Bucket.put(bucket, "milk", 1)
  assert KV.Bucket.get(bucket, "milk") == 1
end
```

может провалиться на этой строке:

```elixir
{:ok, bucket} = KV.Registry.lookup(registry, "shopping")
```

Как эта строка может провалиться, если мы только что создали корзину на предыдущей строке?

Причина этих провалов в том, что мы допустили две ошибки:

  1. Мы сделали оптимизацию (с помощью добавления этого слоя кеширования)
  2. Мы используем `cast/2` (тогда как следует использовать `call/2`)

## Race conditions?

Developing in Elixir does not make your code free of race conditions. However, Elixir's abstractions where nothing is shared by default make it easier to spot a race condition's root cause.

What is happening in our tests is that there is a delay in between an operation and the time we can observe this change in the ETS table. Here is what we were expecting to happen:

1. We invoke `KV.Registry.create(registry, "shopping")`
2. The registry creates the bucket and updates the cache table
3. We access the information from the table with `KV.Registry.lookup(registry, "shopping")`
4. The command above returns `{:ok, bucket}`

However, since `KV.Registry.create/2` is a cast operation, the command will return before we actually write to the table! In other words, this is happening:

1. We invoke `KV.Registry.create(registry, "shopping")`
2. We access the information from the table with `KV.Registry.lookup(registry, "shopping")`
3. The command above returns `:error`
4. The registry creates the bucket and updates the cache table

To fix the failure we need to make `KV.Registry.create/2` synchronous by using `call/2` rather than `cast/2`. This will guarantee that the client will only continue after changes have been made to the table. Let's change the function and its callback as follows:

```elixir
def create(server, name) do
  GenServer.call(server, {:create, name})
end

def handle_call({:create, name}, _from, {names, refs}) do
  case lookup(names, name) do
    {:ok, pid} ->
      {:reply, pid, {names, refs}}
    :error ->
      {:ok, pid} = KV.BucketSupervisor.start_bucket()
      ref = Process.monitor(pid)
      refs = Map.put(refs, ref, name)
      :ets.insert(names, {name, pid})
      {:reply, pid, {names, refs}}
  end
end
```

We changed the callback from `handle_cast/2` to `handle_call/3` and changed it to reply with the pid of the created bucket. Generally speaking, Elixir developers prefer to use `call/2` instead of `cast/2` as it also provides back-pressure - you block until you get a reply. Using `cast/2` when not necessary can also be considered a premature optimization.

Let's run the tests once again. This time though, we will pass the `--trace` option:

```bash
$ mix test --trace
```

The `--trace` option is useful when your tests are deadlocking or there are race conditions, as it runs all tests synchronously (`async: true` has no effect) and shows detailed information about each test. This time we should be down to one or two intermittent failures:

```
  1) test removes buckets on exit (KV.RegistryTest)
     test/kv/registry_test.exs:19
     Assertion with == failed
     code: KV.Registry.lookup(registry, "shopping") == :error
     lhs:  {:ok, #PID<0.109.0>}
     rhs:  :error
     stacktrace:
       test/kv/registry_test.exs:23
```

According to the failure message, we are expecting that the bucket no longer exists on the table, but it still does! This problem is the opposite of the one we have just solved: while previously there was a delay between the command to create a bucket and updating the table, now there is a delay between the bucket process dying and its entry being removed from the table.

Unfortunately this time we cannot simply change `handle_info/2`, the operation responsible for cleaning the ETS table, to a synchronous operation. Instead we need to find a way to guarantee the registry has processed the `:DOWN` notification sent when the bucket crashed.

An easy way to do so is by sending a synchronous request to the registry: because messages are processed in order, if the registry replies to a request sent after the `Agent.stop` call, it means that the `:DOWN` message has been processed. Let's do so by creating a "bogus" bucket, which is a synchronous request, after `Agent.stop` in both tests:


```elixir
  test "removes buckets on exit", %{registry: registry} do
    KV.Registry.create(registry, "shopping")
    {:ok, bucket} = KV.Registry.lookup(registry, "shopping")
    Agent.stop(bucket)

    # Do a call to ensure the registry processed the DOWN message
    _ = KV.Registry.create(registry, "bogus")
    assert KV.Registry.lookup(registry, "shopping") == :error
  end

  test "removes bucket on crash", %{registry: registry} do
    KV.Registry.create(registry, "shopping")
    {:ok, bucket} = KV.Registry.lookup(registry, "shopping")

    # Stop the bucket with non-normal reason
    Agent.stop(bucket, :shutdown)

    # Do a call to ensure the registry processed the DOWN message
    _ = KV.Registry.create(registry, "bogus")
    assert KV.Registry.lookup(registry, "shopping") == :error
  end
```

Our tests should now (always) pass!

This concludes our optimization chapter. We have used ETS as a cache mechanism where reads can happen from any processes but writes are still serialized through a single process. More importantly, we have also learned that once data can be read asynchronously, we need to be aware of the race conditions it might introduce.

In practice, if you find yourself in a position where you need a process registry for dynamic processes, you should use [the `Registry` module](https://hexdocs.pm/elixir/Registry.html) provided as part of Elixir. It provides functionality similar to the one we have built using a GenServer + `:ets` while also being able to perform both writes and reads concurrently. [It has been benchmarked to scale across all cores even on machines with 40 cores](https://elixir-lang.org/blog/2017/01/05/elixir-v1-4-0-released/).

Next let's discuss external and internal dependencies and how Mix helps us manage large codebases.

