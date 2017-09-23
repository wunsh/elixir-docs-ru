---
title: Динамический супервизор
next_page: mix-otp/ets
prev_page: mix-otp/supervisor-and-application
---

# {{ page.title }}

Мы успешно определили наш супервизор, который автоматически запускается (и останавливается) как часть жизненного цикла нашего приложения.

Remember however that our `KV.Registry` is both linking (via `start_link`) and monitoring (via `monitor`) bucket processes in the `handle_cast/2` callback:

Вспомните, однако, что наш `KV.Registry` одновременно и связывает (через `start_link`) и мониторит (через `monitor`) процессы корзин в обратном вызове `handle_cast/2`:

```elixir
{:ok, pid} = KV.Bucket.start_link([])
ref = Process.monitor(pid)
```

Ссылки двунаправлены, а значит падение корзины приведёт к падению всего реестра. Хотя у нас есть супервизор, который гарантирует, что реестр восстановит свою работу, падение реестра всё ещё приведёт к потере всех данных о связи имён корзин с их процессами.

Другими словами, мы хотим, чтобы реестр продолжал работать, даже если корзина падает. Давайте напишем новый тест для реестра:

```elixir
test "removes bucket on crash", %{registry: registry} do
  KV.Registry.create(registry, "shopping")
  {:ok, bucket} = KV.Registry.lookup(registry, "shopping")

  # Stop the bucket with non-normal reason
  Agent.stop(bucket, :shutdown)
  assert KV.Registry.lookup(registry, "shopping") == :error
end
```

Тест похож на "удаление корзин при выходе", кроме того, что мы передаём немного более жесткий вариант для выхода: `:shutdown` вместо `:normal`. Если процесс прекращает жизнь с причиной, отличной от `:normal`, все связанные процессы получают сигнал EXIT, что приводит к прекращению их всех, кроме случая, если они избегают выхода.

С завершением работы корзины, реестр тоже отключается, и наш тест падает при попытке вызвать `GenServer.call/3`:

```
  1) test removes bucket on crash (KV.RegistryTest)
     test/kv/registry_test.exs:26
     ** (exit) exited in: GenServer.call(#PID<0.148.0>, {:lookup, "shopping"}, 5000)
         ** (EXIT) no process: the process is not alive or there's no process currently associated with the given name, possibly because its application isn't started
     code: assert KV.Registry.lookup(registry, "shopping") == :error
     stacktrace:
       (elixir) lib/gen_server.ex:770: GenServer.call/3
       test/kv/registry_test.exs:33: (test)
```

Мы решим эту проблему, определив новый супервизор, который будет порождать все корзины и отслеживать их состояние. Есть стратегия супервизора, которая называется `:simple_one_for_one`, и она прекрасно подходит для таких ситуаций: она позволяет нам задать шаблон воркера и отслеживать множество потомков, основанных на этом шаблоне. С этой стратегией ни один воркер не запускается во время инициализации супервизора. Вместо этого, они запускаются вручную с помощью `Supervisor.start_child/2`.

## Супервизор корзин

Давайте определим наш `KV.BucketSupervisor` в `lib/kv/bucket_supervisor.ex` как показано ниже:

```elixir
defmodule KV.BucketSupervisor do
  use Supervisor

  # A simple module attribute that stores the supervisor name
  @name KV.BucketSupervisor

  def start_link(_opts) do
    Supervisor.start_link(__MODULE__, :ok, name: @name)
  end

  def start_bucket do
    Supervisor.start_child(@name, [])
  end

  def init(:ok) do
    Supervisor.init([KV.Bucket], strategy: :simple_one_for_one)
  end
end
```

Здесь есть два отличия от супервизора, который мы сделали сначала.

Во-первых, мы решили дать супервизору локальное имя `KV.BucketSupervisor`. Мы могли посылать `opts`, полученный в `start_link/1`, в супервизор, но для простоты мы задали имя прямо в коде. Помните, что такой подход имеет свои минусы. Например, мы не сможем запустить несколько экземпляров `KV.BucketSupervisor` во время тестов, они будут конфликтовать по имени. В этом случае, нам стоит позволить всем реестрам использовать один супервизор корзин, и это не будет проблемой, если потомки супервизора `:simple_one_for_one` не взаимодействуют друг с другом.

Мы также определили функцию `start_bucket/0`, которая запускает корзины, как потомков нашего супервизора `KV.BucketSupervisor`. `start_bucket/0` - это функция, которую мы будем вызывать вместо прямого вызова `KV.Bucket.start_link/1` в реестре.

Запустите `iex -S mix`, чтобы попробовать наш новый супервизор:

```iex
iex> {:ok, _} = KV.BucketSupervisor.start_link([])
{:ok, #PID<0.70.0>}
iex> {:ok, bucket} = KV.BucketSupervisor.start_bucket
{:ok, #PID<0.72.0>}
iex> KV.Bucket.put(bucket, "eggs", 3)
:ok
iex> KV.Bucket.get(bucket, "eggs")
3
```

Мы почти готовы к использованию этого супервизора в нашем приложении. Первый шаг - изменить реестр, используя вызов `start_bucket`:

```elixir
  def handle_cast({:create, name}, {names, refs}) do
    if Map.has_key?(names, name) do
      {:noreply, {names, refs}}
    else
      {:ok, pid} = KV.BucketSupervisor.start_bucket()
      ref = Process.monitor(pid)
      refs = Map.put(refs, ref, name)
      names = Map.put(names, name, pid)
      {:noreply, {names, refs}}
    end
  end
```

The second step is to make sure `KV.BucketSupervisor` is started when our application boots. We can do this by opening `lib/kv/supervisor.ex` and changing `init/1` to the following:

Второй шаг - убедиться, что `KV.BucketSupervisor` запускается при загрузке приложения. Мы можем сделать это, открыв `lib/kv/supervisor.ex` и изменив `init/1` следующим образом:

```elixir
  def init(:ok) do
    children = [
      {KV.Registry, name: KV.Registry},
      KV.BucketSupervisor
    ]

    Supervisor.init(children, strategy: :one_for_one)
  end
```

Этого достаточно, чтобы наши тесты проходили, но в нашем приложении есть утечка ресурсов. Когда корзина экстренно завершает работу, супервизор запускает новую на её месте. В конце концов, в этом и заключается роль супервизора!

Однако, когда супервизор перезапускает корзину, реестр не знает об этом. Поэтому у нас будет пустая корзина в супервизоре, к которой никто не может получить доступ! Чтобы решить это, мы хотим указать, что корзины на самом деле временные. Если они падают, независимо от причины, они не должны быть перезапущены.

Мы можем сделать это, передав опцию `restart: :temporary` в строке `use Agent`, которая находится в `KV.Bucket`:

```elixir
defmodule KV.Bucket do
  use Agent, restart: :temporary
```

Давайте также добавим тест в `test/kv/bucket_test.exs`, который будет гарантировать, что корзина является временной.

```elixir
  test "are temporary workers" do
    assert Supervisor.child_spec(KV.Bucket, []).restart == :temporary
  end
```

Наш тест использует функцию `Supervisor.child_spec/2`, чтобы получить спецификацию потомка из модуля, затем устанавливает значение перезапуска в `:temporary`. Сейчас вы можете задаться вопросом, зачем вообще использовать супервизор, если он никогда не перезапускает своих потомков. Это нужно, потому что супервизоры осуществляют не только перезапуск, они также гарантируют корректный запуск и отключение, особенно в случае падений в дереве супервизора.

## Деревья супервизора

Когда мы добавили `KV.BucketSupervisor` в качестве потомка `KV.Supervisor`, мы получили ситуацию, когда супервизоры отслеживают состояние других супервизоров, формируя так называемые "деревья супервизоров".

Каждый раз, когда вы добавляете нового потомка супервизору, важно убедиться, что выбрана правильная стратегия супервизора, а также порядок процессов-потомков. В данном случае мы используем `:one_for_one` и `KV.Registry` запускается до `KV.BucketSupervisor`.

Первый недостаток - проблема правильного порядка. Если `KV.Registry` вызывает `KV.BucketSupervisor`, тогда `KV.BucketSupervisor` должен запускаться раньше `KV.Registry`. В противном случае может произойти так, что реестр попытается обратиться к супервизору корзин до того, как он будет запущен.

Второй недостаток связан со стратегией супервизора. Если `KV.Registry` умирает, вся информация, связывающая имена `KV.Bucket` с процессами корзин, будет потеряна. Кроме того, `KV.BucketSupervisor` и все его потомки должны будут также завершить работу - иначе у нас останутся "осиротевшие" процессы.

В свете этих подробностей, нам стоит рассмотреть альтернативные стратегии супервизора. Два других варианта - `:one_for_all` и `:rest_for_one`. Супервизор, использующий `:rest_for_one`, будет перезапускать процессы-потомки, которые были запущены *после* упавшего потомка. В таком случае мы бы хотели, чтобы `KV.BucketSupervisor` завершился, если `KV.Bucket` завершается. Также при этом нужно поместить супервизор корзин после реестра. А это приведёт к проблеме порядка, которую мы обнаружили двумя абзацами ранее.

Таким образом у нас остался всего один вариант, на который все надежды - стратегия `:one_for_all`: супервизор будет перезапускать всех потомков, при падении любого из них. Это имеет смысл в нашем приложении, т.к. реестр не может работать без супервизора корзин, и супервизор корзин не должен работать без реестра. Давайте переделаем `init/1` в `KV.Supervisor`, включив данные свойства:

```elixir
  def init(:ok) do
    children = [
      KV.BucketSupervisor,
      {KV.Registry, name: KV.Registry}
    ]

    Supervisor.init(children, strategy: :one_for_all)
  end
```

Чтобы помочь разработчикам запомнить, как работают супервизоры и их удобные функции, [Benjamin Tan Wei Hao](http://benjamintan.io/) создал [Supervisor cheat sheet](https://raw.githubusercontent.com/benjamintanweihao/elixir-cheatsheets/master/Supervisor_CheatSheet.pdf).

И у нас осталось ещё две темы для обсуждения перед тем, как мы перейдём к следующей главе.

## Общее состояние в тестах

Недавно мы начали использовать один реестр для одного теста, чтобы быть уверенными в их изолированности:

```elixir
setup do
  {:ok, registry} = start_supervised(KV.Registry)
  %{registry: registry}
end
```

Т.к. теперь мы изменили наш реестр для использования `KV.BucketSupervisor`, который доступен глобально, наши тесты полагаются на этот общий супервизор, хотя каждый тест имеет свой реестр. Вопрос в том, правильно ли это?

Ответ неоднозначен. Нормально полагаться на общее состояние, пока мы зависим только от непересекающихся частей этого состояния. Хотя несколько реестров могут запускать корзины в общем супервизоре корзин, эти корзины и реестры изолированы друг от друга. Мы можем столкнуться только с проблемами параллельного запуска, если будем использовать функции вроде `Supervisor.count_children(KV.Bucket.Supervisor)`, которые будут считать все корзины во всех реестрах, потенциально давая нам разные результаты при параллельном запуске тестов.

Т.к. мы пока зависим только от непересекающихся частей супервизора корзин, нам нет смысла беспокоиться о проблемах с параллельным запуском набора тестов. Если это когда-нибудь станет проблемой, мы можем запускать супервизор для каждого теста и передавать его аргументом в функцию реестра `start_link`.

## Observer

Now that we have defined our supervision tree, it is a great opportunity to introduce the Observer tool that ships with Erlang. Start your application with `iex -S mix` and key this in:

```iex
iex> :observer.start
```

A GUI should pop-up containing all sorts of information about our system, from general statistics to load charts as well as a list of all running processes and applications.

In the Applications tab, you will see all applications currently running in your system along side their supervision tree. You can select the `kv` application to explore it further:

<img src="/images/contents/kv-observer.png" width="640" alt="Observer GUI screenshot" />

Not only that, as you create new buckets on the terminal, you should see new processes spawned in the supervision tree shown in Observer:

```iex
iex> KV.Registry.create KV.Registry, "shopping"
:ok
```

We will leave it up to you to further explore what Observer provides. Note you can double click any process in the supervision tree to retrieve more information about it, as well as right-click a process to send "a kill signal", a perfect way to emulate failures and see if your supervisor reacts as expected.

At the end of the day, tools like Observer is one of the reasons you want to always start processes inside supervision trees, even if they are temporary, to ensure they are always reachable and introspectable.

Now that our buckets are properly linked and supervised, let's see how we can speed things up.
