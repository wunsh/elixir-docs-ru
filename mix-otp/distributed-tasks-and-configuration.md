---
title: Распределенные задачи и конфигурация
---

# {{ page.title }}

В последней главе мы вернёмся к приложению `:kv` и добавим слой маршрутизации, который позволит нам распределять задачи между узлами, основываясь на имени корзины.

Слой маршрутизации будет получать таблицу маршрутизации в следующем формате:

```elixir
[{?a..?m, :"foo@computer-name"},
 {?n..?z, :"bar@computer-name"}]
```

Маршрутизатор будет искать первый байт имени корзины в таблице и передавать запрос нужному узлу. Например, если имя начинается с буквы "a" (`?a` представляет код буквы "a" в Юникоде), запрос будет передан узлу `foo@computer-name`.

Если подходящий узел тот, что обрабатывает запрос, мы закончили маршрутизацию, и узел выполнит запрошенную операцию. Если же подошел другой узел, мы отправим запрос этому узлу, который будет проверять свою таблицу маршрутизации (которая может отличаться от той, что была на первом узле) и действовать аналогично. Если ни один узел не подошел, будет выброшена ошибка.

Вы можете удивиться, почему мы не запрашиваем у найденного в таблице узла выполнение запроса напрямую, а вместо этого посылаем запрос на дальнейшую маршрутизацию этому узлу. Пока таблица маршрутизации такая простая, как показано выше, её логично использовать для всех узлов, но пересылка запросов маршрутизации позволяет гораздо проще разделить таблицу маршрутизации на небольшие части, когда приложение начинает расти. Возможно в какой-то момент `foo@computer-name` будет отвечать только за маршрутизацию запросов к хрализищу, а корзины будут храниться на разных узлах. При этом `bar@computer-name` не должен ничего знать о таких изменениях.

> Замечание: Мы будем использовать оба узла на одной машине в этой главе. Вы можете использовать две (или больше) разных машины в одной сети, но для этого нужно будет сделать некоторые приготовления. Для начала нужно убедиться, что на машинах есть файл `~/.erlang.cookie` с одинаковым значением. Далее необходимо, чтобы [epmd](http://www.erlang.org/doc/man/epmd.html) был запущен на незаблокированном порту (вы можете запустить `epmd -d` для получения отладочной информации). И наконец, если вы хотите больше узнать о распределенности, мы рекомендуем [замечательную главу "Distribunomicon" из "Learn You Some Erlang"](http://learnyousomeerlang.com/distribunomicon).

## Наш первый распределённый код

В Эликсире из коробки есть возможность подключать узлы и обмениваться информацией между ними. Фактически мы используем ту же концепцию, что и с процессами, сообщения отправляются и принимаются в распределённом окружении, потому что процессы Эликсира имеют *прозрачное расположение*. Это значит, что при отправке сообщения нам неважно, находится процесс-получатель на этом или на другом узле, виртуальная машина доставит его в любом случае.

Чтобы запустить распределённый код, нам нужно запустить виртуальную машину, задав ей имя. Имя может быть коротким (при расположении узлов в одной сети) или длинным (включающим полный адрес машины). Запустим новую сессию IEx:

```bash
$ iex --sname foo
```

Вы можете увидеть, что приглашение строки ввода отличается и показывает имя узла после имени машины:

    Interactive Elixir - press Ctrl+C to exit (type h() ENTER for help)
    iex(foo@jv)1>

Мой компьютер назван `jv`, поэтому я вижу `foo@jv` в примере выше, но вы получите другой результат. Мы будем использовать `foo@computer-name` в дальнейших примерах, а вам нужно будет изменить их в соответствии с именем своей машины для запуска кода.

Объявим модуль `Hello` в этом терминале:

```iex
iex> defmodule Hello do
...>   def world, do: IO.puts "hello world"
...> end
```

Если у вас есть другой компьютер в той же сети с установленными Эрлангом и Эликсиром, вы можете запустить на нём вторую сессию IEx. Если нет, запустите её в другом терминале. В обоихслучаях дайте ей короткое имя `bar`:

```bash
$ iex --sname bar
```

Обратите внимание, что внутри этой новой сессии у нас нет доступа к `Hello.world/0`:

```iex
iex> Hello.world
** (UndefinedFunctionError) undefined function: Hello.world/0
    Hello.world()
```

Однако, мы можем порождать новые процессы на `foo@computer-name`, находясь на `bar@computer-name`! Попробуйте (только укажите вместо `@computer-name` то имя, которое видите у себя):

```iex
iex> Node.spawn_link :"foo@computer-name", fn -> Hello.world end
#PID<9014.59.0>
hello world
```

Эликсир породил процесс на другом узле и вернул его pid. Затем код выполнился на другом узле, где существует функция `Hello.world/0` и вызвал эту функцию. Обратите внимание, что результат "hello world" был выведен на текущем узле `bar`, но не на `foo`. Другими словами, сообщение было отправлено назад с `foo` на `bar`. Это происходит, потому что процесс, порождённый на другом узле (`foo`) всё ещё имеет лидера группы на текущем узле (`bar`). Мы кратко говорили о лидерах групп в [главе IO](/getting-started/io-and-the-file-system.html#processes-and-group-leaders).

Мы также можем посылать и принимать сообщения на pid, возвращённый `Node.spawn_link/2`, как обычно. Попробуем простой ping-pong пример:

```iex
iex> pid = Node.spawn_link :"foo@computer-name", fn ->
...>   receive do
...>     {:ping, client} -> send client, :pong
...>   end
...> end
#PID<9014.59.0>
iex> send pid, {:ping, self()}
{:ping, #PID<0.73.0>}
iex> flush()
:pong
:ok
```

Из нашего небольшого исследования мы можем заключить, что нужно использовать `Node.spawn_link/2` для порождения процессов на удалённом узле каждый раз, когда нам нужны распределенные вычисления. Однако, мы также выяснили в этом руководстве, что порождение процессов вне дерева супервизора следует максимально избегать, поэтому нам нужно найти другие варианты.

Есть три лучших альтернативы `Node.spawn_link/2`, которые можно использовать в нашем примере:

1. Можно использовать модуль [:rpc](http://www.erlang.org/doc/man/rpc.html) из Эрланга для выполнения функций на удалённом узле. В оболочке `bar@computer-name` выше, вы можете вызвать `:rpc.call(:"foo@computer-name", Hello, :world, [])` и получите "hello world"

2. Можно запустить сервер на другом узле и посылать запросы через [GenServer](https://hexdocs.pm/elixir/GenServer.html) API. Например, можно осуществить вызов к другому серверу через `GenServer.call({name, node}, arg)` или отправку PID удалённого процесса в качестве первого аргумента

3. Можно исользовать [задачи](https://hexdocs.pm/elixir/Task.html), которые мы изучили в [предыдущей главе](/getting-started/mix-otp/task-and-gen-tcp.html), т.к. они могут быть порождены и на локальном, и на удалённом узле

Варианты выше имеют свои особенности. `:rpc` и использование GenServer сериализуют запрос к одному серверу, тогда как задачи будут запущены асинфронно на удалённом узле, сериализация будет произведена только на уровне порождения супервизором.

На нашем уровне маршрутизации мы будем использовать задачи, но вы также можете попробовать остальные альтернативы.

## async/await

До сих пор мы использовали задачи, которые запускаются и работают в изоляции, игнорируя возвращаемые ими значения. Однако, иногда полезно запустить задачу для вычисления значения и прочитать этот результат после. Для этого в задачах есть шаблон `async/await`:

```elixir
task = Task.async(fn -> compute_something_expensive end)
res  = compute_something_else()
res + Task.await(task)
```

`async/await` предоставляет очень простой механизм параллельного вычисления значений. Кроме того, `async/await` может быть использован с тем же [`Task.Supervisor`](https://hexdocs.pm/elixir/Task.Supervisor.html), который мы использовали в предыдущих главах. Достаточно вызвать `Task.Supervisor.async/2` вместо `Task.Supervisor.start_child/2` и использовать `Task.await/2` для чтения результата.

## Распределённые задачи

Распределённые задачи - ровно то же самое, что контроллируемые супервизором задачи. Единственная разиница в том, что мы передаём имя узла супервизору при порождении задачи. Откройте `lib/kv/supervisor.ex` из приложения `:kv`. Давайте добавим супервизор задач как последнего потомка в дереве:

```elixir
{Task.Supervisor, name: KV.RouterTasks},
```

Теперь запустим два именованных узла снова, но внутри приложения `:kv`:

```bash
$ iex --sname foo -S mix
$ iex --sname bar -S mix
```

Изнутри `bar@computer-name` мы теперь можем порождать задачи прямо на другом узле через супервизор:

```iex
iex> task = Task.Supervisor.async {KV.RouterTasks, :"foo@computer-name"}, fn ->
...>   {:ok, node()}
...> end
%Task{owner: #PID<0.122.0>, pid: #PID<12467.88.0>, ref: #Reference<0.0.0.400>}
iex> Task.await(task)
{:ok, :"foo@computer-name"}
```

Наша первая распределённая задача получает имя узла, на котором запускать задачу. Обратите внимание, что мы передали анонимную функцию в `Task.Supervisor.async/2`, но для распределённых случаев предпочтительно передавать модуль, функцию и аргументы явно:

```iex
iex> task = Task.Supervisor.async {KV.RouterTasks, :"foo@computer-name"}, Kernel, :node, []
%Task{owner: #PID<0.122.0>, pid: #PID<12467.89.0>, ref: #Reference<0.0.0.404>}
iex> Task.await(task)
:"foo@computer-name"
```

Разница в том, что анонимная функция обязывает иметь одинаковый код на узле выполнения и узле, который осуществляет вызов. Использование модуля, функции и аргументов более надёжный вариант, вам достаточно найти функцию, которая подходит по арности в переданном модуле.

С этими знаниями можно наконец написать код маршрутизации.

## Routing layer

Create a file at `lib/kv/router.ex` with the following contents:

```elixir
defmodule KV.Router do
  @doc """
  Dispatch the given `mod`, `fun`, `args` request
  to the appropriate node based on the `bucket`.
  """
  def route(bucket, mod, fun, args) do
    # Get the first byte of the binary
    first = :binary.first(bucket)

    # Try to find an entry in the table() or raise
    entry =
      Enum.find(table(), fn {enum, _node} ->
        first in enum
      end) || no_entry_error(bucket)

    # If the entry node is the current node
    if elem(entry, 1) == node() do
      apply(mod, fun, args)
    else
      {KV.RouterTasks, elem(entry, 1)}
      |> Task.Supervisor.async(KV.Router, :route, [bucket, mod, fun, args])
      |> Task.await()
    end
  end

  defp no_entry_error(bucket) do
    raise "could not find entry for #{inspect bucket} in table #{inspect table()}"
  end

  @doc """
  The routing table.
  """
  def table do
    # Replace computer-name with your local machine name.
    [{?a..?m, :"foo@computer-name"},
     {?n..?z, :"bar@computer-name"}]
  end
end
```

Let's write a test to verify our router works. Create a file named `test/kv/router_test.exs` containing:

```elixir
defmodule KV.RouterTest do
  use ExUnit.Case, async: true

  test "route requests across nodes" do
    assert KV.Router.route("hello", Kernel, :node, []) ==
           :"foo@computer-name"
    assert KV.Router.route("world", Kernel, :node, []) ==
           :"bar@computer-name"
  end

  test "raises on unknown entries" do
    assert_raise RuntimeError, ~r/could not find entry/, fn ->
      KV.Router.route(<<0>>, Kernel, :node, [])
    end
  end
end
```

The first test invokes `Kernel.node/0`, which returns the name of the current node, based on the bucket names "hello" and "world". According to our routing table so far, we should get `foo@computer-name` and `bar@computer-name` as responses, respectively.

The second test checks that the code raises for unknown entries.

In order to run the first test, we need to have two nodes running. Move into `apps/kv` and let's restart the node named `bar` which is going to be used by tests.

```bash
$ iex --sname bar -S mix
```

And now run tests with:

```bash
$ elixir --sname foo -S mix test
```

The test should pass.

## Test filters and tags

Although our tests pass, our testing structure is getting more complex. In particular, running tests with only `mix test` causes failures in our suite, since our test requires a connection to another node.

Luckily, ExUnit ships with a facility to tag tests, allowing us to run specific callbacks or even filter tests altogether based on those tags. We have already used the `:capture_log` tag in the previous chapter, which has its semantics specified by ExUnit itself.

This time let's add a `:distributed` tag to `test/kv/router_test.exs`:

```elixir
@tag :distributed
test "route requests across nodes" do
```

Writing `@tag :distributed` is equivalent to writing `@tag distributed: true`.

With the test properly tagged, we can now check if the node is alive on the network and, if not, we can exclude all distributed tests. Open up `test/test_helper.exs` inside the `:kv` application and add the following:

```elixir
exclude =
  if Node.alive?, do: [], else: [distributed: true]

ExUnit.start(exclude: exclude)
```

Now run tests with `mix test`:

```bash
$ mix test
Excluding tags: [distributed: true]

.......

Finished in 0.1 seconds (0.1s on load, 0.01s on tests)
7 tests, 0 failures, 1 skipped
```

This time all tests passed and ExUnit warned us that distributed tests were being excluded. If you run tests with `$ elixir --sname foo -S mix test`, one extra test should run and successfully pass as long as the `bar@computer-name` node is available.

The `mix test` command also allows us to dynamically include and exclude tags. For example, we can run `$ mix test --include distributed` to run distributed tests regardless of the value set in `test/test_helper.exs`. We could also pass `--exclude` to exclude a particular tag from the command line. Finally, `--only` can be used to run only tests with a particular tag:

```bash
$ elixir --sname foo -S mix test --only distributed
```

You can read more about filters, tags and the default tags in [`ExUnit.Case` module documentation](https://hexdocs.pm/ex_unit/ExUnit.Case.html).

## Application environment and configuration

So far we have hardcoded the routing table into the `KV.Router` module. However, we would like to make the table dynamic. This allows us not only to configure development/test/production, but also to allow different nodes to run with different entries in the routing table. There is a feature of  <abbr title="Open Telecom Platform">OTP</abbr> that does exactly that: the application environment.

Each application has an environment that stores the application's specific configuration by key. For example, we could store the routing table in the `:kv` application environment, giving it a default value and allowing other applications to change the table as needed.

Open up `apps/kv/mix.exs` and change the `application/0` function to return the following:

```elixir
def application do
  [extra_applications: [:logger],
   env: [routing_table: []],
   mod: {KV, []}]
end
```

We have added a new `:env` key to the application. It returns the application default environment, which has an entry of key `:routing_table` and value of an empty list. It makes sense for the application environment to ship with an empty table, as the specific routing table depends on the testing/deployment structure.

In order to use the application environment in our code, we need to replace `KV.Router.table/0` with the definition below:

```elixir
@doc """
The routing table.
"""
def table do
  Application.fetch_env!(:kv, :routing_table)
end
```

We use `Application.fetch_env!/2` to read the entry for `:routing_table` in `:kv`'s environment. You can find more information and other functions to manipulate the app environment in the [Application module](https://hexdocs.pm/elixir/Application.html).

Since our routing table is now empty, our distributed test should fail. Restart the apps and re-run tests to see the failure:

```bash
$ iex --sname bar -S mix
$ elixir --sname foo -S mix test --only distributed
```

The interesting thing about the application environment is that it can be configured not only for the current application, but for all applications. Such configuration is done by the `config/config.exs` file. For example, we can configure IEx default prompt to another value. Just open `apps/kv/config/config.exs` and add the following to the end:

```elixir
config :iex, default_prompt: ">>>"
```

Start IEx with `iex -S mix` and you can see that the IEx prompt has changed.

This means we can also configure our `:routing_table` directly in the `apps/kv/config/config.exs` file:

```elixir
# Replace computer-name with your local machine nodes.
config :kv, :routing_table,
       [{?a..?m, :"foo@computer-name"},
        {?n..?z, :"bar@computer-name"}]
```

Restart the nodes and run distributed tests again. Now they should all pass.

Since Elixir v1.2, all umbrella applications share their configurations, thanks to this line in `config/config.exs` in the umbrella root that loads the configuration of all children:

```elixir
import_config "../apps/*/config/config.exs"
```

The `mix run` command also accepts a `--config` flag, which allows configuration files to be given on demand. This could be used to start different nodes, each with its own specific configuration (for example, different routing tables).

Overall, the built-in ability to configure applications and the fact that we have built our software as an umbrella application gives us plenty of options when deploying the software. We can:

* deploy the umbrella application to a node that will work as both TCP server and key-value storage

* deploy the `:kv_server` application to work only as a TCP server as long as the routing table points only to other nodes

* deploy only the `:kv` application when we want a node to work only as storage (no TCP access)

As we add more applications in the future, we can continue controlling our deploy with the same level of granularity, cherry-picking which applications with which configuration are going to production.

You can also consider building multiple releases with a tool like [Distillery](https://github.com/bitwalker/distillery), which will package the chosen applications and configuration, including the current Erlang and Elixir installations, so we can deploy the application even if the runtime is not pre-installed on the target system.

Finally, we have learned some new things in this chapter, and they could be applied to the `:kv_server` application as well. We are going to leave the next steps as an exercise:

* change the `:kv_server` application to read the port from its application environment instead of using the hardcoded value of 4040

* change and configure the `:kv_server` application to use the routing functionality instead of dispatching directly to the local `KV.Registry`. For `:kv_server` tests, you can make the routing table point to the current node itself

## Summing up

In this chapter, we have built a simple router as a way to explore the distributed features of Elixir and the Erlang <abbr title="Virtual Machine">VM</abbr>, and learned how to configure its routing table. This is the last chapter in our Mix and  <abbr title="Open Telecom Platform">OTP</abbr> guide.

Throughout the guide, we have built a very simple distributed key-value store as an opportunity to explore many constructs like generic servers, supervisors, tasks, agents, applications and more. Not only that, we have written tests for the whole application, got familiar with ExUnit, and learned how to use the Mix build tool to accomplish a wide range of tasks.

If you are looking for a distributed key-value store to use in production, you should definitely look into [Riak](http://basho.com/riak/), which also runs in the Erlang <abbr title="Virtual Machine">VM</abbr>. In Riak, the buckets are replicated, to avoid data loss, and instead of a router, they use [consistent hashing](https://en.wikipedia.org/wiki/Consistent_hashing) to map a bucket to a node. A consistent hashing algorithm helps reduce the amount of data that needs to be migrated when new nodes to store buckets are added to your infrastructure.

Happy coding!
