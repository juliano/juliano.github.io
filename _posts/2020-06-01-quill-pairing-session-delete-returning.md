---
layout: single
title: "Quill pairing session: DELETE... RETURNING"
header:
  teaser: /assets/images/galaxy-hole.jpg
  imgcredit: Image by DarkWorkX from Pixabay
categories:
  - scala
  - quill
  - opensource
  - howto
classes: wide
---

Recently, [someone has opened an issue suggesting the support for `DELETE ... RETURNING` clause](https://github.com/getquill/quill/issues/1863){:target="_blank"}. This post will take you through the joy of implementing it!

## Pairing session 2.0 - the mission

[Different from last time](/2019/11/18/contributing-to-quill-a-pairing-session/), we won't be starting from scratch, figuring what needs to be done. Mostly because I've recently worked on [update returning](/2020/03/25/quill-update-returning/), the knowledge is fresh in my mind, which makes it easy to implement this feature.

We will be pairing, but this time I assume your knowledge on Quill's codebase. The plan is simple:

- Change Quill's DSL
- Make sure it translate properly to SQL

Let's dive into the code.

## Introducing `delete.returning` to [`QueryDsl`](https://github.com/getquill/quill/blob/master/quill-core/src/main/scala/io/getquill/dsl/QueryDsl.scala){:target="_blank"}

This is easy, we just need to add a method `returning` to `Delete`, which will return an `ActionReturning`:

```scala
sealed trait Delete[E] extends Action[E] {
  @compileTimeOnly(NonQuotedException.message)
  def returning[R](f: E => R): ActionReturning[E, R] = NonQuotedException()
}
```

Before testing, [the knowledge acquired from `update returning`](https://github.com/getquill/quill/pull/1720/files#diff-d1be86fd3dd0aaa20dd13a7e82c40f66){:target="blank"} tells us to add `QueryDsl#Delete` to [Parsing](https://github.com/getquill/quill/blob/master/quill-core/src/main/scala/io/getquill/quotation/Parsing.scala#L919){:target="blank"}:

```scala
// line 919
private def reprocessReturnClause(ident: Ident, originalBody: Ast, action: Tree) = {
    val actionType = typecheckUnquoted(action)

    (ident == originalBody, actionType.tpe) match {
      // Note, tuples are also case classes so this also matches for tuples
      case (true, ClassTypeRefMatch(cls, List(arg))) if (cls == asClass[QueryDsl#Insert[_]] || cls == asClass[QueryDsl#Update[_]] || cls == asClass[QueryDsl#Delete[_]]) && isTypeCaseClass(arg) =>
      /// ...
```

When a `.returning` clause returns the initial record i.e. `.returning(r => r)`, we need to expand out the record into it's fields i.e. `.returning(r => (r.foo, r.bar))`. You obviously knew that ;)

Let's ensure everything behaves as expected with some tests. `delete` doesn't take parameters, so a few of them (a subset of the tests for `update`) will cover the possibilities in [`ActionMacroSpec`](https://github.com/getquill/quill/blob/master/quill-core/src/test/scala/io/getquill/context/ActionMacroSpec.scala){:target="_blank"}:

```scala
"delete returning" - {
  "returning value" in {
    val q = quote {
      qr1.delete.returning(t => t.l)
    }
    val r = testContext.run(q)
    r.string mustEqual """querySchema("TestEntity").delete.returning((t) => t.l)"""
    r.prepareRow mustEqual Row()
    r.returningBehavior mustEqual ReturnRecord
  }
  "returning value - with single - should not compile" in testContext.withDialect(MirrorIdiomReturningSingle) { ctx =>
    "import ctx._; ctx.delete.returning(t => t.l))" mustNot compile
  }
  "returning value - with multi" in testContext.withDialect(MirrorIdiomReturningMulti) { ctx =>
    import ctx._
    val q = quote {
      qr1.delete.returning(t => t.l)
    }
    val r = ctx.run(q)
    r.string mustEqual """querySchema("TestEntity").delete.returning((t) => t.l)"""
    r.prepareRow mustEqual Row()
    r.returningBehavior mustEqual ReturnColumns(List("l"))
  }
}
```

## Changing [`SqlIdiom`](https://github.com/getquill/quill/blob/master/quill-sql-portable/src/main/scala/io/getquill/sql/idiom/SqlIdiom.scala){:target="_blank"}

There are actually two possible outcomes from the new `delete.returning`, depending on the `Dialect`:

- `PostgresDialect` will generate the SQL `DELETE FROM table RETURNING field1, field2`
- `SQLServerDialect` will generate the SQL `DELETE FROM table OUTPUT DELETED.field1, DELETED.field2`

The former seems easier, we should look at it first.

### "standard" `DELETE RETURNING`

The first change we made in `QueryDsl` already covers the common case. We can see that adding the following test to [SqlActionMacroSpec](https://github.com/getquill/quill/blob/master/quill-sql/src/test/scala/io/getquill/context/sql/SqlActionMacroSpec.scala){:target="_blank"}:

```scala
"returning clause - multi" in testContext.withDialect(MirrorSqlDialectWithReturnClause) { ctx =>
  import ctx._
  val q = quote {
    qr1.delete.returning(r => (r.i, r.l))
  }
  val mirror = ctx.run(q)
  mirror.string mustEqual "DELETE FROM TestEntity RETURNING i, l"
  mirror.returningBehavior mustEqual ReturnRecord
}
```

However, something more complex, like returning with a query, fails:

```scala
"with returning clause - query" - {
  case class Dummy(i: Int)

  "simple with filter using id" in testContext.withDialect(MirrorSqlDialectWithReturnClause) { ctx =>
    import ctx._
    val q = quote {
      qr1.delete.returning(r => query[Dummy].filter(d => d.i == r.i).max)
    }
    val mirror = ctx.run(q)
    mirror.string mustEqual "DELETE FROM TestEntity AS r RETURNING (SELECT MAX(*) FROM Dummy d WHERE d.i = r.i)"
    mirror.returningBehavior mustEqual ReturnRecord
  }
}
```

The resulting `String` is actually:

```scala
"DELETE FROM TestEntity RETURNING (SELECT MAX(*) FROM Dummy d WHERE d.i = r.i)"
```

Which is close, but incorrect. It lacks the alias for `r`. It happens because of the way [SqlIdiom handles `Delete` at the moment](https://github.com/getquill/quill/blob/master/quill-sql-portable/src/main/scala/io/getquill/sql/idiom/SqlIdiom.scala#L425){:target="_blank"}:

```scala
// line 425
protected def actionTokenizer(insertEntityTokenizer: Tokenizer[Entity])(implicit astTokenizer: Tokenizer[Ast], strategy: NamingStrategy): Tokenizer[Action] =
  Tokenizer[Action] {
    // ...
    case Delete(Filter(table: Entity, x, where)) =>
      stmt"DELETE FROM ${table.token} WHERE ${where.token}"

    case Delete(table: Entity) =>
      stmt"DELETE FROM ${table.token}"
```

It generates simple `DELETE` statements, without creating aliases - it hasn't been necessary up to this point. Let's make sure the alias generation logic is in place:

```scala
protected def actionTokenizer(insertEntityTokenizer: Tokenizer[Entity])(implicit astTokenizer: Tokenizer[Ast], strategy: NamingStrategy): Tokenizer[Action] =
  Tokenizer[Action] {
    // ...
    case Delete(Filter(table: Entity, x, where)) =>
      stmt"DELETE FROM ${table.token}${actionAlias.map(alias => stmt" AS ${alias.token}").getOrElse(stmt"")} WHERE ${where.token}"

    case Delete(table: Entity) =>
      stmt"DELETE FROM ${table.token}${actionAlias.map(alias => stmt" AS ${alias.token}").getOrElse(stmt"")}"
```

This is the only change we need, all the heavy lifting has already been in place. We must cover it [adding a bunch of tests to `SqlActionMacroSpec`](https://github.com/getquill/quill/blob/ffc58e40a87cf98fca9b1e40108ec5b6ccf56075/quill-sql/src/test/scala/io/getquill/context/sql/SqlActionMacroSpec.scala#L914){:target="_blank"}, and another bunch to [`postgres/JdbcContextSpec`](https://github.com/getquill/quill/blob/ffc58e40a87cf98fca9b1e40108ec5b6ccf56075/quill-jdbc/src/test/scala/io/getquill/context/jdbc/postgres/JdbcContextSpec.scala#L235){:target="_blank"}.

### `DELETE RETURNING` via `OUTPUT`

[Quill supports returning via `OUTPUT`](/2019/12/05/quill-sql-server-returning-via-output/) for SQL Server, so the new feature needs to support it as well.

Once again, the heavy lifting has already been there. Having that in mind, we need to include `Delete` in the `SqlIdiom#actionTokenizer` pattern matching and add the logic to generate a query result with `OUTPUT DELETED.fieldName` - a slightly variance of `Update`:

```scala
// line 458
case r @ ReturningAction(action, alias, prop) =>
  idiomReturningCapability match {
    // ...
    case OutputClauseSupported => action match {
      // ...
      case Delete(_) =>
        stmt"${action.token} OUTPUT ${returnListTokenizer.token(ExpandReturning(r, Some("DELETED"))(this, strategy).map(_._1))}"
```

Last but not least, the new code needs test coverage:

- [tests added to `SqlActionMacroSpec`](https://github.com/getquill/quill/blob/2ada4b5ceb4a16c283747512a72627c342f48c70/quill-sql/src/test/scala/io/getquill/context/sql/SqlActionMacroSpec.scala#L1063){:target="_blank"}
- [tests added to `sqlserver/JdbcContextSpec`](https://github.com/getquill/quill/blob/2ada4b5ceb4a16c283747512a72627c342f48c70/quill-jdbc/src/test/scala/io/getquill/context/jdbc/sqlserver/JdbcContextSpec.scala#L149){:target="_blank"}

## That was quick!

Indeed, given our knowledge of the codebase, it wasn't necessary to explore and navigate the code [as much as last time](/2019/11/18/contributing-to-quill-a-pairing-session/), most of the time we could reuse solutions already in place. The long nights studying Quill's codebase paid off!

If you are curious about the change as a whole, [have a look at the pull request](https://github.com/getquill/quill/pull/1870){:target="_blank"}.
