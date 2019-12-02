---
layout: single
title: "Quill NDBC Postgres: A New Async Module"
header:
  teaser: /assets/images/puzzle-dna.jpg
  imgcredit: Image by Arek Socha from Pixabay
categories:
  - scala
  - quill
  - ndbc
  - postgres
  - opensource
---

[Quill 3.5.0 is out](https://twitter.com/deusaquilus/status/1200310094128390147) and this release brings an integration with the [already mentioned NDBC Driver](/2019/11/02/the-journey-of-an-open-source-developer/#there-and-back-again). This post is a brief summary about the new module `quill-ndbc-postgres`.

## Motivation

Quill has an asynchronous module called `quill-async` which provides support to MySQL and Postgres databases. This module is built on top of [mauricio/postgresql-async](https://github.com/mauricio/postgresql-async): a [Netty](https://netty.io/) based, database drivers for Postgres and MySQL written in Scala. It is a great solution, and we have been using it for a while now.

But all of sudden, [this issue appeared](https://github.com/getquill/quill/issues/1178):

<a href="https://getquill.io/">
![](/assets/images/issue-async-deprecated.png){: .align-center}
</a>

Wait... what?

<a href="https://getquill.io/">
![](/assets/images/mauricio-async-deprecated.png){: .align-center}
</a>

Well, it happens. A replacement is necessary.

## NDBC: Non-blocking Database Connectivity

[NDBC's goal is to provide a full asyncronous approach to handle databases](https://ndbc.io/). It is built using Netty and [Trane.io Futures](http://trane.io), a High-performance Future implementation for the JVM. Sounds's like a good replacement!

Took me some time to implement it, but a few days ago [my pull request integrating NDBC got merged](https://github.com/getquill/quill/pull/1702). Now we have an alternative to `quill-async-postgres`; async support via NDBC driver is available with Postgres database.

## Quill + NDBC

To replace the deprecated driver just a few changes are necessary. First, add `quill-ndbc-postgres` to your dependencies:

```scala
libraryDependencies ++= Seq(
  "io.getquill" %% "quill-ndbc-postgres" % "3.4.11"
)
```

The `application.properties` needs to change as well:

```properties
ctx.ndbc.dataSourceSupplierClass=io.trane.ndbc.postgres.netty4.DataSourceSupplier
ctx.ndbc.host=database-host
ctx.ndbc.port=1234
ctx.ndbc.user=username
ctx.ndbc.password=password
ctx.ndbc.database=dbname
```

The main difference is the first line, NDBC needs to know which data source supplier class to use, for Postgres it is `io.trane.ndbc.postgres.netty4.DataSourceSupplier`.

> Even though NDBC is written on top of Netty, we keep the api and implementation apart. This way we can build another implementation, on top of [Akka](https://akka.io/), for instance.

Now we have everything in place to start using `NdbcPostgresContext`:

```scala
val ctx = new NdbcPostgresContext(Literal, "ctx")
```

That's all, fairly straightforward. When you start using it, let me know how it goes for you in the comments, or please [open an issue at the project's github](https://github.com/getquill/quill/issues).

The next step is the MySQL implementation. I will let you know when it happens :)
