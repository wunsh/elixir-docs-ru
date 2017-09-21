---
title: ETS
next_page: mix-otp/dependencies-and-umbrella-apps
prev_page: mix-otp/dynamic-supervisor
---

Каждый раз, когда нам нужно обратиться к корзине, мы вынуждены отправлять сообщение в реестр. В случае, если к реестру конкуретно обращаются несколько процессов, он рискует стать узким местом!

В этой главе мы поговорим о ETS (Erlang Term Storage, хранилище термов Эрланга), и о том, как его использовать для кеширования

> Осторожно! Не начинайте использовать ETS для кеша преждевременно! Логируйте и анализируйте производительность вашего приложения и ищите узкие места, чтобы знать, *нужен ли* вам кеш, и *что* вам нужно кешировать. Эта глава - просто пример, как можно использовать ETS, когда вы поймёте, что это вам действительно нужно.

## ETS для кеша

ETS allows us to store any Elixir term in an in-memory table. Working with ETS tables is done via [Erlang's `:ets` module](http://www.erlang.org/doc/man/ets.html):

ETS позволяет нам хранить любой терм Эликсира в таблице в памяти. Работа с таблицами ETS возможна через [модуль Эрланга `:ets`](http://www.erlang.org/doc/man/ets.html):

```elixir
iex> table = :ets.new(:buckets_registry, [:set, :protected])
8207
iex> :ets.insert(table, {"foo", self()})
true
iex> :ets.lookup(table, "foo")
[{"foo", #PID<0.41.0>}]
```

Для создания таблицы ETS обязательны два аргумента: имя таблицы и набор опций. Из доступных опций мы передали тип таблицы и правила доступа к ней. Мы выбрали тип `:set`, который запрещает дублирование ключей. Мы также обозначили доступ к таблице как `:protected`, чтобы позволить записывать данные в таблицу только процессу, который её создал, а все остальные могут только читать эти данные. Эти значения стандартные, поэтому далее мы их будем опускать.

Таблицы ETS могут быть также именованными, позволяя получать доступ к ним, передавая имя:

```elixir
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

## Состояния гонки?

Разработка на Эликсире не гарантирует, что в коде не будет состояний гонки. Однако, абстракции Эликсира, где нет общих данных по умолчанию облегчанает поиск причин возникновения состояния гонки.

Во время выполнения наших тестах происходит задержка между операциями и есть время, когда мы можем посмотреть на изменения в таблице ETS. Вот что мы ожидаем увидеть:

1. Мы выполняем `KV.Registry.create(registry, "shopping")`
2. Реестр создаёт корзину и обновляет таблицу кеша
3. Мы смотрим информацию из таблицы с помощью `KV.Registry.lookup(registry, "shopping")`
4. Команда выше возвращает `{:ok, bucket}`

Однако, т.к. `KV.Registry.create/2` - это операция преобразования (cast operation), команда закончится до того, как мы на самом деле запишем данные в таблицу! Другими словами, вот что произойдёт на самом деле:

1. Мы выполняем `KV.Registry.create(registry, "shopping")`
2. Мы получаем доступ к информации из таблицы с помощью `KV.Registry.lookup(registry, "shopping")`
3. Команда выше возвращает `:error`
4. Реестр создаёт корзину и обновляет таблицу кеша

Чтобы исправить эту проблему, нам нужно сделать `KV.Registry.create/2` синхронной с помощью `call/2` вместо `cast/2`. Это гарантирует, что клиент продолжит выполнение только после изменений, сделанных в таблице. Давайте изменим функцию и её колбэк, как показано ниже:

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

Мы изменили коллбэк с `handle_cast/2` на `handle_call/3` и изменили его, чтобы возвращался PID созданной корзины. Говоря в целом, Эликсир разработчики предпочитают использовать `call/2` вместо `cast/2`, потому что это также блокирует выполнение до получения ответа. Использование `cast/2`, когда это не нужно может быть воспринято как преждевреманная оптимизация.

Давайте запустим тесты ещё раз. Однако теперь мы передадим опцию `--trace`:

```bash
$ mix test --trace
```

Опция `--trace` полезна для выполнения выявления дедлоков или состояния гонки, т.к. все тесты запускаются синхронно (`async: true` не имеет эффекта) и показывает детальную информацию о каждом тесте. В этот раз мы должны получить одну или две (intermittent) ошибки:

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

Следуя сообщениям об ошибках, мы ожидаем, что корзина больше не существует в таблице, но это не так! Эта проблема обратна той, которую мы только что решили: тогда как ранее была задержка между командой создания корзины и обновлением таблицы, теперь есть задержка между смертью процесса корзины и удалением из неё записи.

К несчастью, в этот раз мы не можем просто изменить `handle_info/2`, операцию, ответственную за очистку таблицы ETS, для выполнения синхронно. Напротив, нам нужно найти способ отправки реестром уведомления `:DOWN`, когда корзина падает.

Лёгкий способ сделать это - отправлять синхронно запрос к реестру: потому что сообщения приходят по очереди, если реестр отвечает на запрос после вызова `Agent.stop`, это значит, что сообщение `:DOWN` было отправлено. Давайте сделаем так, создав корзину "bogus" синхронным запросом, после `Agent.stop` в обоих тестах:


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

Теперь все тесты должны пройти, и должны проходить всегда.

На этом мы заканчиваем главу об оптимизации. Мы использовали ETS для кеша, в котором чтение может быть выполнено любым процессом, а запись только одним процессом. Более важно то, мы узнали, что данные могут быть прочитанны асинхронно, а значит нам нужно опасаться возникновения состояния гонки.

На практике, если вам понадобится реестр для динамических процессов, посмотрите на [модуль `Registry`](https://hexdocs.pm/elixir/Registry.html), который является частью Эликсира. Он предоставляет функциональность, подобную той, которую мы сделали с помощью GenServer и `:ets`, а также позволяет выполнять чтение и запись параллельно. [Это было испытано бенчмарком на линейную масштабируемость даже на машине с 40 ядрами](https://elixir-lang.org/blog/2017/01/05/elixir-v1-4-0-released/).

Далее давайте поговорим о внешних и внутренних зависимостях, и о том, как Mix помогает управлять большими базами исходников.
