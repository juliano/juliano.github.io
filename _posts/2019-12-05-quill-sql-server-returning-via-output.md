---
layout: single
title: "Quill SQL Server: Returning via OUTPUT"
header:
  teaser: /assets/images/harvest.jpg
  imgcredit: Image by Susanne Jutzeler, suju-foto from Pixabay
categories:
  - scala
  - quill
  - sqlserver
  - opensource
---

Besides [NDBC Postgres module](/2019/11/29/quill-ndbc-postgres-a-new-async-module), the release of Quill 3.5.0 brings an interesting feature for Microsoft SQL Server users: returning via OUTPUT.
![](/assets/images/sql-server-logo.png){: .align-center}

## The `returning` feature

Quill has the `returning` feature, which behaves in different ways, according to the limitations of each database. Postgres has the largest set of functionalities around `returning`, given that Postgres dialect supports the [`INSERT ... RETURNING`](https://www.postgresql.org/docs/9.6/sql-insert.html) clause:

```scala
case class Product(id: Int, description: String, sku: Long)

val q = quote {
  query[Product].insert(lift(Product(1, "My Product", 1011L))).returning(r => (r.id, r.description))
}

ctx.run(q)
// INSERT INTO Product (id, description, sku) VALUES (?, ?, ?) RETURNING id, description
```

Quite useful. SQL Server supports a very similar feature [via `OUTPUT` clause](https://docs.microsoft.com/en-us/sql/t-sql/queries/output-clause-transact-sql?view=sql-server-2017). The 3.5.0 release brings this feature to Quill as well.

> While implementing this feature I've found the inspiration to write the post [Contributing to Quill, a Pairing Session](/2019/11/18/contributing-to-quill-a-pairing-session/), so if you are interested in how this feature came to life, there you go :)

## `INSERT ... OUTPUT`

The way to use the new feature in SQL Server is exactly the same:

```scala
val q = quote {
  query[Product].insert(lift(Product(1, "SQL Server", 1433L))).returning(r => (r.id, r.description))
}

ctx.run(q)
// INSERT INTO Product (id, description, sku) OUTPUT INSERTED.id, INSERTED.description VALUES (?, ?, ?)
```

As mentioned before, it is more limited than Postgres `returning`. While we can even query the return clause in Postgres, the best we can do on SQL Server are arithmetic operations:

```scala
val q = quote {
  query[Product].insert(lift(Product(1, "SQL Server", 1433L)))
}

ctx.run(q.returning(r => r.id + 42))
// INSERT INTO Product (id, description, sku) OUTPUT INSERTED.id + 42 VALUES (?, ?, ?)
```

This is a limitation of SQL Server, so there's not much Quill can do about it - except for [making sure that invalid operations don't compile](https://github.com/getquill/quill/blob/master/quill-sql/src/test/scala/io/getquill/context/sql/SqlActionMacroSpec.scala#L692).

Do you use Quill with SQL Server? Give us feedback about the new feature!
