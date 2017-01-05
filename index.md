#Главная

Elixir - это динамический функциональный язык, спроектированный для создания масштабируемых и легко поддерживаемых приложений.

Elixir использует виртуальную машину Erlang, известную по быстродействующим, распределённым и отказоустойчивым системам, которые хорошо зарекомендовали себя как в веб-разработке, так и в сфере встроенного ПО.

Чтобы узнать больше об Elixir, обратите внимание на руководство для новичков. Или продолжите чтение, чтобы получить представление о платформе, языке и инструментах

#Особенности платформы

##Масштабируемость

Весь код на Elixir запускается внутри легковесных потоков выполнения (называемых процессами), которые изолированы и обмениваются информацией через сообщения:

```
current_process = self()

# Порождение процесса Elixir (который не является процессом операционной системы!)
spawn_link(fn ->
  send current_process, {:msg, "hello world"}
end)

# Блокировка до принятия сообщения
receive do
  {:msg, contents} -> IO.puts contents
end
```

Благодаря их легковесной природе, не редко на одной машине одновременно выполняются сотни тысяч процессов. Изоляция позволяет сборщику мусора работать с ними независимо, уменьшая задержки всей системы, и используя все ресурсы системы эффективно, насколько это возможно (вертикальное масштабирование).

Процессы также могут взаимодействовать с другими процессами, запущенными на других машинах в одной сети. Это обеспечивает основу для распределения, позволяющего разработчикам согласовать работу между несколькими узлами (горизотальное масштабирование).

##Отказоустойчивость

Неизбежная правда о ПО, запущенном в продакшне, заключается в том, что ошибки обязательно случаются. Тем более, когда мы начинаем использовать сеть, файловые системы и другие сторонние ресурсы.

Для работы со сбоями Elixir предоставляет супервизоры, которые описывают, как перезапускать части вашей системы, когда всё пошло наперекосяк, возвращаясь к известному начальному состоянию, которое гарантированно работает:

```
import Supervisor.Spec

children = [
  supervisor(TCP.Pool, []),
  worker(TCP.Acceptor, [4040])
]

Supervisor.start_link(children, strategy: :one_for_one)
```

#Особенности языка

##Функциональное программирование

Функциональное программирование диктует стиль кодирования, который помогает разрабочтикам писать короткий, быстрый и поддерживаемый код. Например, сравнение c шаблоном (pattern matching) позволяет разработчикам легко дестуктурировать данные и получить доступ к их содержимому:

```
%User{name: name, age: age} = User.get("John Doe")
name #=> "John Doe"
```

Совмещённое с ограничивающими условиями сравнение с шаблоном позволяет нам элегантно сравнивать и добалять условия для выполнения некоторого кода:

```
def serve_drinks(%User{age: age}) when age >= 21 do
  # Code that serves drinks!
end

serve_drinks User.get("John Doe")
#=> Fails if the user is under 21
```

Elixir опирается на эти особенности, чтобы обеспечить работу вашего ПО в рамках ожидаемых ограничений. И, если что-то всё равно пойдёт нет так, не беспокойтесь, спервизоры вернут всё назад!

##Расширяемость и DSL

Elixir - расширяемый язык. Он позволяет разработчикам расширять язык для конкретных областей, чтобы повысить продуктивность работы.

В качестве примера, напишем простой тест, использующий тестовый фрэймворк Elixir’а, который называется ExUnit:

```
defmodule MathTest do
  use ExUnit.Case, async: true

  test "can add two numbers" do
    assert 1 + 1 == 2
  end
end
```

The async: true option allows tests to run in parallel, using as many CPU cores as possible, while the assert functionality can introspect your code, providing great reports in case of failures. Those features are built using Elixir macros, making it possible to add new constructs as if they were part of the language itself.

#Tooling features

##A growing ecosystem

Elixir ships with a great set of tools to ease development. Mix is a build tool that allows you to easily create projects, manage tasks, run tests and more:

```
$ mix new my_app
$ cd my_app
$ mix test
.

Finished in 0.04 seconds (0.04s on load, 0.00s on tests)
1 tests, 0 failures
```

Mix is also able to manage dependencies and integrates nicely with the Hex package manager, which provides dependency resolution and the ability to remotely fetch packages.

##Interactive development

Tools like IEx (Elixir’s interactive shell) are able to leverage many aspects of the language and platform to provide auto-complete, debugging tools, code reloading, as well as nicely formatted documentation:

```
$ iex
Interactive Elixir - press Ctrl+C to exit (type h() ENTER for help)
iex> c "my_file.ex"        # Компилирует файл
iex> t Enum                # Выводит типы, объявленные в модуле Enum
iex> h IEx.pry             # Выводит документацию для IEx pry
iex> i "Hello, World"      # Выводит информацию об указанном типе данных
```

##Erlang compatible

Elixir runs on the Erlang VM giving developers complete access to Erlang’s ecosystem, used by companies like Heroku, WhatsApp, Klarna, Basho and many more to build distributed, fault-tolerant applications. An Elixir programmer can invoke any Erlang function with no runtime cost:

```
iex> :crypto.hash(:md5, "Using crypto from Erlang OTP")
<<192, 223, 75, 115, ...>>
```

To learn more about Elixir, check our getting started guide. We also have online documentation available and a Crash Course for Erlang developers.