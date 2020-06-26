---
layout: single
title: "Quill News: Version 3.5.2"
header:
  teaser: /assets/images/news.jpg
  imgcredit: Photo by Markus Winkler on Unsplash
categories:
  - scala
  - quill
  - opensource
---

[Quill 3.5.2 was released last week](https://github.com/getquill/quill/releases/tag/v3.5.2){:target="_blank"}, bringing many bug fixes and new capabilities. It took a little longer than usual releases, but for a good reason: regargind the future of Quill, [@deusaquilus](https://twitter.com/deusaquilus){:target="_blank"} has been doing an amazing job preparing the foundations for the upcoming Dotty implementation.

What does that mean? That Quill will be able to support new features and fixes that can't be properly addressed with the current implementation. [Translation of boolean literals](https://github.com/getquill/quill/issues/1685){:target="_blank"} or [problems with `infix`](https://github.com/getquill/quill/issues/1843){:target="_blank"} are a few examples.

Let's see what is new!

## More modules supporting Scala 2.13

Thanks to [@jilen](https://github.com/jilen){:target="_blank"} the modules `quill-async-mysql`, `quill-async-postgres`, `quill-finagle-mysql`, `quill-cassandra`, `quill-cassandra-monix` and `quill-orientdb` are all available for 2.13.

## quill-jasync is here!

Due to [@Assassin4791](https://github.com/Assassin4791){:target="_blank"} and [@danslapman](https://github.com/danslapman){:target="_blank"} Quill offers a new option for async drivers via [jasync-sql](https://github.com/jasync-sql/jasync-sql){:target="_blank"}.

[Find the (very easy) instruction of how to use `quill-jasync` here.](https://github.com/getquill/quill#quill-jasync){:target="_blank"}

## Better error message when lifting enum types

[@majk-p](https://github.com/majk-p){:target="_blank"} made the error message [significantly more useful](https://github.com/getquill/quill/pull/1803){:target="_blank"}, with a good illustrative example.

## Remove use of `Row#getAnyOption` from `FinaglePostgresDecoders`

This one is an internal improvement that fixed [a bug](https://github.com/getquill/quill/issues/1844){:target="_blank"}. The best description of the problem and solution was given by [j-ostrich](https://github.com/j-ostrich){:target="_blank"} himself in [his pull request](https://github.com/getquill/quill/pull/1848){:target="_blank"}.

## Add translate to NDBC Context

`quill-ndbc` has been lacking `translate`, used to [print queries](https://github.com/getquill/quill#printing-queries){:target="_blank"}. [Not anymore!](https://github.com/getquill/quill/pull/1865){:target="_blank"} :smile:

## Fix SqlServer snake case - OUTPUT i_n_s_e_r_t_e_d.id

Remember the feature [Quill SQL Server: Returning via OUTPUT](/2019/12/05/quill-sql-server-returning-via-output/)? Guess what, [a bug happened](https://github.com/getquill/quill/issues/1768){:target="_blank"}.

Fun fact, in order to properly fix it [I had to touch Quill's AST](https://github.com/getquill/quill/pull/1867){:target="_blank"}, it's a good reference for anyone intending to contribute :octocat:

## Delete returning

You already knew about this one, [we built it together](/2020/06/01/quill-pairing-session-delete-returning/)!

## Quill community is just :heart:

Thank you to all the contributions! The community around Quill is wonderful!

Want to contribute as well? [Join us in the Gitter channel](https://gitter.im/getquill/quill)!
