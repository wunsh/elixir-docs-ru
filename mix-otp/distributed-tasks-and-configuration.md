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

## Слой маршрутизации

Создайте файл `lib/kv/router.ex` со следующим содержимым:

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

Давайте напишем тест, чтобы убедиться, что наш маршрутизатор работает. Создайте файл с именем `test/kv/router_test.exs`, содержащий:

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

Первый тест вызывает функцию `Kernel.node/0`, которая возвращает имя текущего узла, основываясь на именах корзин "hello" и "world". Согласно нашей таблице маршрутизации, мы должны получить `foo@computer-name` и `bar@computer-name` в качестве ответов, соответственно.

Второй тест проверяет, что код падает на вводе неизвестных значений.

Для запуска первого теста нам нужно два запущенных узла. Перейдём в `apps/kv` и перезапустим узел с именем `bar`, чтобы использовать его в тестах.

```bash
$ iex --sname bar -S mix
```

И запустим тест следующим образом:

```bash
$ elixir --sname foo -S mix test
```

Тест должен пройти без ошибок.

## Фильтры тестов и тэги

Хотя наши тесты проходят, структура тестов становится более сложной. А именно, запуск тестов через `mix test` приведёт к ошибкам, т.к. наш тест предусматривает подключение к другому узлу.

К счастью, в ExUnit есть средства для тэгирования тестов, которые позволяют запускать определённые обратные вызовы или даже фильтровать тесты по этим тэгам. Мы уже использовали тэг `:capture_log` в предыдущей главе, семантику которого определяет сам ExUnit.

Теперь давайте добавим тэг `:distributed` в `test/kv/router_test.exs`:

```elixir
@tag :distributed
test "route requests across nodes" do
```

Формулировка `@tag :distributed` эквивалентна `@tag distributed: true`.

Когда тесты отмечены нужными тегами, мы можем проверить, запущен ли узел в сети, и, если нет, мы можем исключить все распределённые тесты. Откройте `test/test_helper.exs` внутри приложения `:kv` и добавьте следующее:

```elixir
exclude =
  if Node.alive?, do: [], else: [distributed: true]

ExUnit.start(exclude: exclude)
```

Теперь запустите `mix test`:

```bash
$ mix test
Excluding tags: [distributed: true]

.......

Finished in 0.1 seconds (0.1s on load, 0.01s on tests)
7 tests, 0 failures, 1 skipped
```

На этот раз все тесты прошл и ExUnit предупреждает нас, что распределённые тесты были исключены. Если вы запустите `$ elixir --sname foo -S mix test`, один дополнительный тест должен запуститься и проходить, пока узел `bar@computer-name` доступен.

Команда `mix test` также позволяет нам динамически включать и исключать теги. Например, мы можем запустить `$ mix test --include distributed` для запуска распределенных тестов независимо от значения в `test/test_helper.exs`. Мы также можем передать `--exclude` чтобы исключить тег. Наконец, `--only` можно использовать для запуска только тестов с определённым тегом:

```bash
$ elixir --sname foo -S mix test --only distributed
```

Вы можете прочитать больше о фильтрах, тегах и стандартных тегах в [документации модуля `ExUnit.Case`](https://hexdocs.pm/ex_unit/ExUnit.Case.html).

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

## Заключение

В этой главе мы создали простой маршрутизатор, чтобы попробовать на практике возможности распределённой работы Эликсира и виртуальной машины Эрланга, а также изучили конфигурирование таблиц маршрутизации. Это последняя глава в нашем руководстве по Mix и <abbr title="Open Telecom Platform">OTP</abbr>

В этом руководстве мы сделали очень простое распределённое хранилище пар ключ-значение, чтобы исследовать многие конструкции, например, GenServer, супервизоры, задачи, агенты, приложения и другие. Кроме того, мы написали тесты для всего нашего приложения, познакомились с ExUnit, и изучили использование средства сборки Mix для решения многих задач.

Если вы ищете распределённое хранилище ключ-значение для продакшна, вам определённо стоит посмотреть [Riak](http://basho.com/riak/), который также работает в виртуальной машине Эрланга. В Riak корзины реплицируются во избежание потери данных, и, в отличие от маршрутизатора, они используют [консистентное хеширование](https://ru.wikipedia.org/wiki/%D0%9A%D0%BE%D0%BD%D1%81%D0%B8%D1%81%D1%82%D0%B5%D0%BD%D1%82%D0%BD%D0%BE%D0%B5_%D1%85%D0%B5%D1%88%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%B8%D0%B5) для сопоставления корзины и узла сети. Алгоритм консистентного хеширования помогает уменьшить количество данных, которые нужно мигрировать, когда в инфраструктуру добавляется новый узел для хранения корзин.

Приятного кодинга!