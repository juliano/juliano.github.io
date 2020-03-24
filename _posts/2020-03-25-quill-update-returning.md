---
layout: single
title: "Quill: Update... Returning"
header:
  teaser: /assets/images/plane-landing.jpg
  imgcredit: Photo by Dries Vinke on Unsplash
categories:
  - scala
  - quill
  - postgres
  - sqlserver
  - opensource
---

The first [Quill's release of 2020 is out](https://github.com/getquill/quill) and besides fixes, it brings some useful features. The first of them is `update(...).returning`.

As you know, [Quill supports `insert(...).returning`](https://github.com/getquill/quill#returning) for Postgres (via `RETURNING`), [and more recently, for SQL Server (via `OUTPUT`)](/2019/12/05/quill-sql-server-returning-via-output/) as well.

There's been an [open issue requesting the `update ... returning` feature quite some time](https://github.com/getquill/quill/issues/572). This feature is now available!

## The `update returning` feature

The new feature works exactly in the same way as `insert(...).returning`, generating a different SQL according to the database. Given a `Product`:

```scala
case class Product(id: Int, description: String, sku: Long)
```

And the following `quote`:

```scala
val q = quote {
  query[Product]
    .update(lift(Product(16, "Awesome Product", 42L)))
    .returning(p => (p.id, p.description))
}
```

When running it using Postgres, the result is:

```scala
val updated = pgCtx.run(q)

// UPDATE Product SET id = ?, description = ?, sku = ?
//   RETURNING id, description
```

And for SQL Server users:

```scala
val updated = sqlServerCtx.run(q)

// UPDATE Product SET id = ?, description = ?, sku = ?
//   OUTPUT INSERTED.id, INSERTED.description
```

Interested in the code changes? [Check the pull request out!](https://github.com/getquill/quill/pull/1720)
