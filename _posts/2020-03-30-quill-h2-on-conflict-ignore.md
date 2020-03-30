---
layout: single
title: "Quill H2: On Conflict Ignore"
header:
  teaser: /assets/images/chess-conflict.jpg
  imgcredit: Photo by Randy Fath on Unsplash
categories:
  - scala
  - quill
  - h2
  - opensource
---

Since [version 1.4.200](http://www.h2database.com/html/changelog.html), H2 supports `INSERT ... ON CONFLICT DO NOTHING` [in PostgreSQL Compatibility Mode](http://www.h2database.com/html/features.html#compatibility). Thanks to [b-gyula](https://github.com/b-gyula), Quill now supports this feature as well.

## `insert(...).onConflictIgnore`

The [full upsert feature](https://github.com/getquill/quill#insert-or-update-upsert-conflict) is supported by Postgres, SQLite and MySQL. The new H2 feature is more limited, allowing only `onConflictIgnore`. Given a `Product` and a simple `quote`:

```scala
import io.getquill._

val ctx = new SqlMirrorContext(H2Dialect, Literal)
import ctx._

case class Product(id: Int, description: String, sku: Long)

val q = quote {
  query[Product].insert(_.id -> 1, _.sku -> 10).onConflictIgnore
}
```

Running the `quote`, it generates:

```scala
ctx.run(q)
// INSERT INTO Product (id,sku) VALUES (1, 10) ON CONFLICT DO NOTHING
```

However, be aware that trying to set a conflict target will fail:

```scala
val q2 = quote {
  query[Product].insert(_.id -> 1, _.sku -> 10).onConflictIgnore(_.id)
}

ctx.run(q)
// java.lang.IllegalStateException: Only onConflictIgnore upsert is supported in H2 (v1.4.200+)
```

Interested in the code changes? [Have a look at b-gyula's pull request](https://github.com/getquill/quill/pull/1731).
