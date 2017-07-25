---
title: Куда двигаться дальше
---

# {{ page.title }}

Хотите изучить Эликсир глубже? Продолжайте читать!

## Сделайте свой первый проект на Эликсире

Для того, чтобы начать ваш первый проект, в поставке Эликсира есть инструмент сборки Mix. Вы можете начать новый проект запустив:

```bash
$ mix new path/to/new/project
```

Мы написали руководство, которое объясняет, как сделать приложение на Эликсире, с его собственным деревом супервизора, конфигурацией, тестами и прочим. Приложение работает с распределенным хранилищем ключ-значение, в котором мы объединяем пары ключ-значение в "корзины" (buckets) и распределяем эти "корзины" между несколькими нодами:

* [Mix и OTP](/getting-started/mix-otp/introduction-to-mix.html)

## Meta-programming

Elixir is an extensible and very customizable programming language thanks to its meta-programming support. Most meta-programming in Elixir is done through macros, which are very useful in several situations, especially for writing DSLs. We have written a short guide that explains the basic mechanisms behind macros, shows how to write macros, and how to use macros to create DSLs:

* [Meta-programming in Elixir](/getting-started/meta/quote-and-unquote.html)

## Community and other resources

We have a [Learning](/learning.html) section that suggests books, screencasts, and other resources for learning Elixir and exploring the ecosystem. There are also plenty of Elixir resources out there, like conference talks, open source projects, and other learning material produced by the community.

Don't forget that you can also check the [source code of Elixir itself](https://github.com/elixir-lang/elixir), which is mostly written in Elixir (mainly the `lib` directory), or [explore Elixir's documentation](/docs.html).

## A byte of Erlang

Elixir runs on the Erlang Virtual Machine and, sooner or later, an Elixir developer will want to interface with existing Erlang libraries. Here's a list of online resources that cover Erlang's fundamentals and its more advanced features:

* This [Erlang Syntax: A Crash Course](/crash-course.html) provides a concise intro to Erlang's syntax. Each code snippet is accompanied by equivalent code in Elixir. This is an opportunity for you to not only get some exposure to Erlang's syntax but also review some of the things you have learned in this guide.

* Erlang's official website has a short [tutorial](http://www.erlang.org/course/concurrent_programming.html) with pictures that briefly describe Erlang's primitives for concurrent programming.

* [Learn You Some Erlang for Great Good!](http://learnyousomeerlang.com/) is an excellent introduction to Erlang, its design principles, standard library, best practices, and much more. Once you have read through the crash course mentioned above, you'll be able to safely skip the first couple of chapters in the book that mostly deal with the syntax. When you reach [The Hitchhiker's Guide to Concurrency](http://learnyousomeerlang.com/the-hitchhikers-guide-to-concurrency) chapter, that's where the real fun starts.