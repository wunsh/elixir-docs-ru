---
title: Документация Elixir
next_page: basic-types
---

Содержание:
* [Базовые типы](/docs/basic-types.html)
* [Базовые операторы](/docs/basic-operators.html)
* [Сопоставление с образцом](/docs/pattern-matching.html)
* [Операторы ветвления](/docs/case-cond-and-if.html)
* [Двоичные данные, строки и списки символов](/docs/binaries-strings-and-char-lists.html)
* [Списки с ключевыми словами и словари](/docs/keywords-and-maps.html)
* [Модули и функции](/docs/modules-and-functions.html)
* [Рекурсия](/docs/recursion.html)
* [Перечисления и потоки](/docs/enumerables-and-streams.html)
* [Процессы](/docs/processes.html)
* [Ввод/вывод и файловая система](/docs/io-and-the-file-system.html)
* [alias, require и import](/docs/alias-require-and-import.html)
* [Атрибуты модулей](/docs/module-attributes.html)
* [Структуры](/docs/structs.html)
* [Протоколы](/docs/protocols.html)
* [Списковые выражения](/docs/comprehensions.html)
* [Сигилы](/docs/sigils.html)
* [try, catch и rescue](/docs/try-catch-and-rescue.html)
* [Спецификации типов и поведения](/docs/typespecs-and-behaviours.html)
* [Библиотеки Эрланга](/docs/erlang-libraries.html)
* [Куда двигаться дальше](/docs/where-to-go-next.html)
* [Введение в Mix](/docs/introduction-to-mix.html)
* [Агент](/docs/agent.html)
* [GenServer](/docs/genserver.html)
* [Супервизор и Приложение](/docs/supervisor-and-application.html)
* [Динамический супервизор](/docs/dynamic-supervisor.html)
* [ETS](/docs/ets.html)
* [Зависимости и зонтичные проекты](/docs/dependencies-and-umbrella-apps.html)
* [Task и gen_tcp](/docs/task-and-gen-tcp.html)
* [Доктесты, паттерны и with](/docs/docs-tests-and-with.html)
* [Распределенные задачи и конфигурация](/docs/distributed-tasks-and-configuration.html)
* [Quote и unquote](/docs/quote-and-unquote.html)
* [Макросы](/docs/macros.html)
* [Предметно-ориентированные языки](/docs/domain-specific-languages.html)

Добро пожаловать!

В этом руководстве мы научим вас основам Эликсира, синтаксису языка, расскажем, как объявлять модули, как управлять основными структурами данных и ещё некоторым вещам. Эта глава посвящена проверке установки Эликсира и запуску его интерактивной оболочки, которая называется IEx.

Требования:

* Elixir - Версия 1.4.0 и выше
* Erlang - Version 18.0 и выше

Приступим!

> Если вы нашли ошибку в этом руководстве, пожалуйста, [сообщите о ней или пришлите исправление](https://github.com/wunsh/elixir-docs-ru).

## Установка

Если вы всё ещё не установили Эликсир, сделайте это с помощью [инструкции по установке](/install). Когда закончите, выполните команду `elixir --version` для получения текущей версии языка.

## Интерактивный режим

Когда вы установите Эликсир, у вас будет три исполняемых команды: `iex`, `elixir` и `elixirc`. Если вы скомпилировали Эликсир из исходников или используете пакетную версию, вы сможете найти их в директории `bin`.

Начнём с выполнения команды `iex` (или `iex.bat` для Виндоус), которая отвечает за запуск интерактивной оболочки Эликсира. В этом режиме мы можем напечатать любое выражение на Эликсире и получить результат его выполнения. Опробуем несколько базовых команд для разогрева.

Откройте `iex` и введите следующие выражения:
```elixir
Erlang/OTP 19 [erts-8.1] [source] [64-bit] [smp:4:4] [async-threads:10] [hipe] [kernel-poll:false] [dtrace]

Interactive Elixir (1.4.0) - press Ctrl+C to exit (type h() ENTER for help)
iex(1)> 40 + 2
42
iex(2)> "hello" <> " world"
"hello world"
```

Учтите, что некоторые детали, такие как номера версий,  могут немного отличаться, и сейчас это не так важно. Далее мы опустим служебный вывод `iex`, чтобы сосредоточиться на коде. Для выхода из `iex` нажмите `Ctrl+C` дважды.

Похоже, можно идти дальше!

> Обратите внимание: если у вас Виндоус, вы также можете попробовать выполнить команду `iex.bat --werl` для получения более удобного интерфейса, в зависимости от консоли, которую вы используете.

## Запуск скриптов

После знакомства с основами языка можно попробовать написать простую программу. Это можно сделать, поместив следующий код на Эликсире в файл:

```elixir
IO.puts "Hello world from Elixir"
```

Сохраните его как `simple.exs` и выполните командой `elixir`:

```bash
$ elixir simple.exs
Hello world from Elixir
```

Позже мы научимся компилировать этот код (в [Главе 8](/docs/modules-and-functions.html)) и использовать инструмент сборки Mix (в [Руководстве по Mix и OTP](/getting-started/mix-otp/introduction-to-mix.html)). Теперь можно двигаться дальше.

## Где можно задать вопрос?

По ходу чтения данного руководства у вас могут возникать вопросы, в конце концов, это часть процесса обучения! Вот несколько мест, где вы можете задать вопрос и узнать больше об Эликсире:

* [Русскоязычный чат в Телеграме](https://t.me/joinchat/BPczSEII11yspp86h0bJeQ)
* [Канал #elixir-lang в IRC](irc://irc.freenode.net/elixir-lang)
* [Эликсир в Слаке](https://elixir-slackin.herokuapp.com/)
* [Форум по Эликсиру](http://elixirforum.com)
* [Тег `elixir` на StackOverflow](https://stackoverflow.com/questions/tagged/elixir)

Чтобы с большей вероятностью получить ответ на ваш вопрос, используйте две подсказки:

* Вместо вопроса "как сделать X в Эликсире", спросите "как решить Y в Эликсире". Другими словами, не стоит спрашивать реализацию конкретного решения, вместо объяснения сути проблемы. Отталкивание от проблемы даст больше контекста для нахождения корректного ответа.

* В случае, если что-то работает не так, как ожидалось, пожалуйста, включите достаточно информации в свой отчёт, например: вашу версию Эликсира, сниппет самого кода и сообщение об ошибке вместе с трассировкой стека. Используйте такие сайты, как [Gist](https://gist.github.com/), для размещения подобной информации.
