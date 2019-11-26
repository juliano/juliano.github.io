---
layout: single
title: "Contributing to Quill, a Pairing Session"
header:
  teaser: /assets/images/quill.png
  imgcredit: Image by Janet Gooch on Pixabay
categories:
  - scala
  - quill
  - opensource
  - howto
classes: wide
---

As one of [Quill](https://github.com/getquill/quill)'s maintainer and an open source enthusiast, I am always thinking of ways to spread the word and have more people contributing to the project. Pair programming is a great technique to share knowledge and introduce newcomers to a project, however, I can't pair with someone in the US, India or Brazil willing to contribute, can I?

Of course I can! Welcome to our your first remote pairing blog session! In this post I will have you as my co-pilot while implementing a new feature to Quill.

![](/assets/images/pairing.jpg){: .align-right}

Think about it as a proper pairing session, where I am talking directly to you, speaking my mind. Sometimes it can feel like I am jumping around the code way too easily, but this is natural in a pairing session where one person has more experience that the other. And don't be ashamed, express your opinions and suggestions in the comments.

## Getting started

The feature we will implement is [Add SQL Server Support for returning via OUTPUT](https://github.com/getquill/quill/issues/1497), meaning it's mostly about `quill-sql` and `quill-jdbc`. Let's walk through this.

Being a SQL Server feature, [`SQLServerDialectSpec`](https://github.com/getquill/quill/blob/e80349e60243e9a8d3d0c0991d77d6c052b1fbb1/quill-sql/src/test/scala/io/getquill/context/sql/idiom/SQLServerDialectSpec.scala#L6) can be a good starting point. However, there aren't any tests regarding `returning`. Still thinking about Sql Server specifically, it's worth checking [`sqlserver/JdbcContextSpec`](https://github.com/getquill/quill/blob/3232763ec5df444382aefcb6c87a6ff2b97fb237/quill-jdbc/src/test/scala/io/getquill/context/jdbc/sqlserver/JdbcContextSpec.scala#L55) as well. There we can find something:

```scala
"Insert with returning with single column table" in {
  val inserted = testContext.run {
    qr4.insert(lift(TestEntity4(0))).returningGenerated(_.i)
  }
  testContext.run(qr4.filter(_.i == lift(inserted))).head.i mustBe inserted
}

"Insert with returning with multiple columns and query embedded" in {
  val inserted = testContext.run {
    qr4Emb.insert(lift(TestEntity4Emb(EmbSingle(0)))).returningGenerated(_.emb.i)
  }
  testContext.run(qr4Emb.filter(_.emb.i == lift(inserted))).head.emb.i mustBe inserted
}
```

Let's try to change these tests to use `.returning`:

```console
[error] .../quill/quill-jdbc/src/test/scala/io/getquill/context/jdbc/sqlserver/JdbcContextSpec.scala:57:49: The 'returning' clause is not supported by the io.getquill.SQLServerDialect idiom. Use 'returningGenerated' instead.
[error]       qr4.insert(lift(TestEntity4(0))).returning(_.i)
```

Maybe we can search for the error message. That takes us to [`Parsing`](https://github.com/getquill/quill/blob/3232763ec5df444382aefcb6c87a6ff2b97fb237/quill-core/src/main/scala/io/getquill/quotation/Parsing.scala#L878):

```scala
idiomReturnCapability match {
  case ReturningMultipleFieldSupported | ReturningClauseSupported =>
  case ReturningSingleFieldSupported =>
    c.fail(s"The 'returning' clause is not supported by the ${currentIdiom.getOrElse("specified")} idiom. Use 'returningGenerated' instead.")
  case ReturningNotSupported =>
    c.fail(s"The 'returning' or 'returningGenerated' clauses are not supported by the ${currentIdiom.getOrElse("specified")} idiom.")
}
```

That definitely doesn't help now, but let's keep it in mind, eventually we will get back to it.

Let's look for something more specific. Maybe [`SQLServerDialect`](https://github.com/getquill/quill/blob/f1c8eb45b86aaa691e1f4017e28457ef13e65504/quill-sql/src/main/scala/io/getquill/SQLServerDialect.scala#15) can tell us something. The trait's signature is:

```scala
trait SQLServerDialect
  extends SqlIdiom
  with QuestionMarkBindVariables
  with ConcatSupport
  with CanReturnField
```

We are getting close, I can feel it! Following `CanReturnField` we will see that it extends [`ReturningCapability`](https://github.com/getquill/quill/blob/f1c8eb45b86aaa691e1f4017e28457ef13e65504/quill-core/src/main/scala/io/getquill/context/ReturnFieldCapability.scala#L3), which has the definitions for all existing returning behaviours.

`ReturningCapability` is returned by `idiomReturningCapability` from trait [`Capabilities`](https://github.com/getquill/quill/blob/f1c8eb45b86aaa691e1f4017e28457ef13e65504/quill-core/src/main/scala/io/getquill/context/ReturnFieldCapability.scala#L39) and its descendents:

```scala
trait Capabilities {
  def idiomReturningCapability: ReturningCapability
}

trait CanReturnClause extends Capabilities {
  override def idiomReturningCapability: ReturningClauseSupported = ReturningClauseSupported
}

trait CanReturnField extends Capabilities {
  override def idiomReturningCapability: ReturningSingleFieldSupported = ReturningSingleFieldSupported
}
...
```

Following `idiomReturningCapability` we will finally find the place where the SQL is generated, in [`SqlIdiom`](https://github.com/getquill/quill/blob/f1c8eb45b86aaa691e1f4017e28457ef13e65504/quill-sql/src/main/scala/io/getquill/context/sql/idiom/SqlIdiom.scala#L426):

```scala
case r @ ReturningAction(Insert(table: Entity, Nil), alias, prop) =>
  idiomReturningCapability match {
    // If there are queries inside of the returning clause we are forced to alias the inserted table (see #1509). Only do this as
    // a last resort since it is not even supported in all Postgres versions (i.e. only after 9.5)
    case ReturningClauseSupported if (CollectAst.byType[Entity](prop).nonEmpty) =>
      SqlIdiom.withActionAlias(this, r)
    case ReturningClauseSupported =>
      stmt"INSERT INTO ${table.token} ${defaultAutoGeneratedToken(prop.token)} RETURNING ${returnListTokenizer.token(ExpandReturning(r)(this, strategy).map(_._1))}"
    case other =>
      stmt"INSERT INTO ${table.token} ${defaultAutoGeneratedToken(prop.token)}"
  }

case r @ ReturningAction(action, alias, prop) =>
  idiomReturningCapability match {
    // If there are queries inside of the returning clause we are forced to alias the inserted table (see #1509). Only do this as
    // a last resort since it is not even supported in all Postgres versions (i.e. only after 9.5)
    case ReturningClauseSupported if (CollectAst.byType[Entity](prop).nonEmpty) =>
      SqlIdiom.withActionAlias(this, r)
    case ReturningClauseSupported =>
      stmt"${action.token} RETURNING ${returnListTokenizer.token(ExpandReturning(r)(this, strategy).map(_._1))}"
    case other =>
      stmt"${action.token}"
  }
```

It starts to get complex. Before we carry on, we should summarize what we've learned so far:

- `SQLServerDialect` extends `CanReturnField`, in order to support `returningGenerated` **only**;
- `CanReturnField` ensures this behaviour, defining that `idiomReturningCapability` returns a `ReturningSingleFieldSupported`;
- `ReturningSingleFieldSupported` is a `ReturningCapability`, mother of all the return behaviours;
- `Parsing` verifies the type of the returning clause and **fails the compilation if necessary**;
- `SqlIdiom` decides how to generate the return statement after checking the current `idiomReturningCapability`

All this acquired knowledge is vital to implement the new feature. Like the Github issue says, we already have a dialect that does what we need, [`PostgresDialect`](https://github.com/getquill/quill/blob/f1c8eb45b86aaa691e1f4017e28457ef13e65504/quill-sql/src/main/scala/io/getquill/PostgresDialect.scala#L10):

```scala
trait PostgresDialect
  extends SqlIdiom
  with QuestionMarkBindVariables
  with ConcatSupport
  with OnConflictSupport
  with CanReturnClause
```

As we know, `CanReturnField.idiomReturnCapability` returns a `ReturningClauseSupported`:

```scala
/**
 * An actual `RETURNING` clause is supported in the SQL dialect of the specified database e.g. Postgres.
 * this typically means that columns returned from Insert/Update/etc... clauses can have other database
 * operations done on them such as arithmetic `RETURNING id + 1`, UDFs `RETURNING udf(id)` or others.
 * In JDBC, the following is done:
 * `connection.prepareStatement(sql, Statement.RETURN_GENERATED_KEYS))`.
 */
sealed trait ReturningClauseSupported extends ReturningCapability
```

`CanReturnClause` is extended by [`MirrorSqlDialectWithReturnClause`](https://github.com/getquill/quill/blob/f1c8eb45b86aaa691e1f4017e28457ef13e65504/quill-sql/src/main/scala/io/getquill/MirrorSqlDialect.scala#L20) as well:

```scala
trait MirrorSqlDialectWithReturnClause
  extends SqlIdiom
  with QuestionMarkBindVariables
  with ConcatSupport
  with CanReturnClause
```

Which is used for tests in [`SqlActionMacroSpec`](https://github.com/getquill/quill/blob/3232763ec5df444382aefcb6c87a6ff2b97fb237/quill-sql/src/test/scala/io/getquill/context/sql/SqlActionMacroSpec.scala#L438), for `returning` and `returningGenerated`:

```scala
"returning clause - single" in testContext.withDialect(MirrorSqlDialectWithReturnClause) { ctx =>
  import ctx._
  val q = quote {
    qr1.insert(lift(TestEntity("s", 0, 1L, None))).returning(_.l)
  }

  val mirror = ctx.run(q)
  mirror.string mustEqual "INSERT INTO TestEntity (s,i,l,o) VALUES (?, ?, ?, ?) RETURNING l"
  mirror.returningBehavior mustEqual ReturnRecord
}
```

Now we know enough to set up a plan of action.

## Development plan

- We need a new `ReturningCapability`, exposed by a new `Capabilities` via `idiomReturningCapability`;
- `SqlIdiom` has to accomodate the new `ReturningCapability` generating the `OUTPUT` clause;
- `Parsing` needs to allow the new code to compile;
- `SQLServerDialect` will extend the new `Capabilities`

And we can deal with the unexpected surprises along the way. The game is on!

### `OutputClauseSupported` and `CanOutputClause`

Following the stablished convention, the names make sense:

```scala
sealed trait OutputClauseSupported extends ReturningCapability

object OutputClauseSupported extends OutputClauseSupported

trait CanOutputClause extends Capabilities {
  override def idiomReturningCapability: OutputClauseSupported = OutputClauseSupported
}
```

Now a new `MirrorSqlDialect` extending `CanOutputClause`:

```scala
trait MirrorSqlDialectWithOutputClause
  extends SqlIdiom
  with QuestionMarkBindVariables
  with ConcatSupport
  with CanOutputClause

object MirrorSqlDialectWithOutputClause extends MirrorSqlDialectWithOutputClause {
  override def prepareForProbing(string: String) = string
}
```

So far so good... or maybe not. This change already breaks the code:

```diff
[error] /Users/juliano.alves/development/opensource/quill/quill-core/src/main/scala/io/getquill/norm/ExpandReturning.scala:23:11: match may not be exhaustive.
[error] It would fail on the following input: OutputClauseSupported
[error]     idiom.idiomReturningCapability match {
[error]           ^
[error] /Users/juliano.alves/development/opensource/quill/quill-core/src/main/scala/io/getquill/quotation/Parsing.scala:831:38: match may not be exhaustive.
[error] It would fail on the following input: OutputClauseSupported
[error]     def verifyAst(returnBody: Ast) = capability match {
[error]                                      ^
[error] /Users/juliano.alves/development/opensource/quill/quill-core/src/main/scala/io/getquill/quotation/Parsing.scala:875:7: match may not be exhaustive.
[error] It would fail on the following input: OutputClauseSupported
[error]       idiomReturnCapability match {
[error]       ^
[error] three errors found
```

Let's take care of `ExpandReturning` first. The error happens because `OutputClauseSupported` is not being taken in consideration during the pattern matching. We are basing the new implementation on `ReturningClauseSupported`, so it's reasonable to simply mirror its behaviour:

```scala
// line 23
idiom.idiomReturningCapability match {
  case ReturningClauseSupported | OutputClauseSupported =>
    ReturnAction.ReturnRecord
  case ReturningMultipleFieldSupported =>
  ...
```

Similar changes in `Parsing`, for both errors:

```scala
// line 831
def verifyAst(returnBody: Ast) = capability match {
  case OutputClauseSupported =>
  case ReturningClauseSupported =>
  // Only .returning(r => r.prop) or .returning(r => OneElementCaseClass(r.prop1..., propN)) or .returning(r => (r.prop1..., propN)) (well actually it's prop22) is allowed.
  case ReturningMultipleFieldSupported =>
    returnBody match {
  ...

// line 875
idiomReturnCapability match {
  case ReturningMultipleFieldSupported | ReturningClauseSupported | OutputClauseSupported =>
  case ReturningSingleFieldSupported =>
    c.fail(s"The 'returning' clause is not supported by the ${currentIdiom.getOrElse("specified")} idiom. Use 'returningGenerated' instead.")
  case ReturningNotSupported =>
    c.fail(s"The 'returning' or 'returningGenerated' clauses are not supported by the ${currentIdiom.getOrElse("specified")} idiom.")
}
```

There is something in this snippet that deserves attention, the `idiomReturnCapability` being matched. It is defined in `Parsing` as well:

```scala
private[getquill] def idiomReturnCapability: ReturningCapability = {
  val returnAfterInsertType =
    currentIdiom
      .toSeq
      .flatMap(_.members)
      .collect {
        case ms: MethodSymbol if (ms.name.toString == "idiomReturningCapability") => Some(ms.returnType)
      }
      .headOption
      .flatten

  returnAfterInsertType match {
    case Some(returnType) if (returnType =:= typeOf[ReturningClauseSupported]) => ReturningClauseSupported
    case Some(returnType) if (returnType =:= typeOf[ReturningSingleFieldSupported]) => ReturningSingleFieldSupported
    case Some(returnType) if (returnType =:= typeOf[ReturningMultipleFieldSupported]) => ReturningMultipleFieldSupported
    case Some(returnType) if (returnType =:= typeOf[ReturningNotSupported]) => ReturningNotSupported
    // Since most SQL Dialects support returing a single field (that is auto-incrementing) allow a return
    // of a single field in the case that a dialect is not actually specified. E.g. when SqlContext[_, _]
    // is used to define `returning` clauses.
    case other => ReturningSingleFieldSupported
  }
}
```

Long story short, `returnAfterInsertType` has the type information about `ReturningCapability` being used by the idiom. Then, the pattern matching returns the corresponding `object` of that type, or a `ReturningSingleFieldSupported` otherwise. So `OutputClauseSupported` has to be included as an option:

```scala
  returnAfterInsertType match {
    case Some(returnType) if (returnType =:= typeOf[ReturningClauseSupported]) => ReturningClauseSupported
    case Some(returnType) if (returnType =:= typeOf[OutputClauseSupported]) => OutputClauseSupported
    ...
```

Code compiling, tests passing. Time to write some tests to the new feature.

## Adapting the SQL Idiom

Let's start simple, adding a test similar to [`returning clause - single`](https://github.com/getquill/quill/blob/master/quill-sql/src/test/scala/io/getquill/context/sql/SqlActionMacroSpec.scala#L438) but generating an `OUTPUT` clause:

```scala
"output clause - single" in testContext.withDialect(MirrorSqlDialectWithOutputClause) { ctx =>
  import ctx._
  val q = quote {
    qr1.insert(lift(TestEntity("s", 0, 1L, None))).returning(_.l)
  }

  val mirror = ctx.run(q)
  mirror.string mustEqual "INSERT INTO TestEntity (s,i,l,o) OUTPUT INSERTED.l VALUES (?, ?, ?, ?)"
  mirror.returningBehavior mustEqual ReturnRecord
}
```

It fails, for obvious reasons. Time to work on `SqlIdiom` and figure how to generate the expected result. Remember [those two pattern clauses related to `idiomReturningCapability`](https://github.com/getquill/quill/blob/f1c8eb45b86aaa691e1f4017e28457ef13e65504/quill-sql/src/main/scala/io/getquill/context/sql/idiom/SqlIdiom.scala#L426)?

```scala
case r @ ReturningAction(Insert(table: Entity, Nil), alias, prop) =>
  idiomReturningCapability match {
  ...

case r @ ReturningAction(action, alias, prop) =>
  idiomReturningCapability match {
  ...
```

In order to identify which of the `ReturningAction`s clauses we should look at, we will start dealing with Quill's ASTs!

## Quill's Abstract Syntax Trees

This is my favourite part of the process, using the console to explore the [AST](https://github.com/getquill/quill/blob/master/quill-core/src/main/scala/io/getquill/ast/Ast.scala). In the `sbt console`:

```scala
scala> import io.getquill._
scala> import io.getquill.ast._
scala> val ctx = new SqlMirrorContext(MirrorSqlDialectWithOutputClause, Literal)
scala> import ctx._

scala> case class TestEntity(s: String, i: Int, l: Long, o: Option[Int])
scala> val qr1 = quote { query[TestEntity] }

scala> val q = quote {
         qr1.insert(lift(TestEntity("s", 0, 1L, None))).returning(_.l)
       }
q: ctx.Quoted[ctx.ActionReturning[TestEntity,Long]]{def quoted: io.getquill.ast.Returning; def ast: io.getquill.ast.Returning; def id380689071(): Unit; val liftings: AnyRef{val TestEntity.apply("s", 0, 1L, scala.None).s: io.getquill.quotation.ScalarValueLifting[String,String]; val TestEntity.apply("s", 0, 1L, scala.None).i: io.getquill.quotation.ScalarValueLifting[Int,Int]; val TestEntity.apply("s", 0, 1L, scala.None).l: io.getquill.quotation.ScalarValueLifting[Long,Long]; val TestEntity.apply("s", 0, 1L, scala.None).o: io.getquill.quotation.ScalarValueLifting[Option[Int],Option[Int]]}} = $anon$1@374e427b
```

To make visualization of the AST easy, Quill integrates the awesome [`PPrint module`](http://www.lihaoyi.com/PPrint/):

```scala
scala> pprint.pprintln(q.ast, 200)
Returning(
  Insert(
    Entity("TestEntity", List()),
    List(
      Assignment(Ident("v"), Property(Ident("v"), "s"), ScalarValueLift("$line9.$read.$iw.$iw.$iw.$iw.$iw.$iw.TestEntity.apply(\"s\", 0, 1L, scala.None).s", "s", MirrorEncoder(<function3>))),
      Assignment(Ident("v"), Property(Ident("v"), "i"), ScalarValueLift("$line9.$read.$iw.$iw.$iw.$iw.$iw.$iw.TestEntity.apply(\"s\", 0, 1L, scala.None).i", 0, MirrorEncoder(<function3>))),
      Assignment(Ident("v"), Property(Ident("v"), "l"), ScalarValueLift("$line9.$read.$iw.$iw.$iw.$iw.$iw.$iw.TestEntity.apply(\"s\", 0, 1L, scala.None).l", 1L, MirrorEncoder(<function3>))),
      Assignment(Ident("v"), Property(Ident("v"), "o"), ScalarValueLift("$line9.$read.$iw.$iw.$iw.$iw.$iw.$iw.TestEntity.apply(\"s\", 0, 1L, scala.None).o", None, MirrorEncoder(<function3>)))
    )
  ),
  Ident("x1"),
  Property(Ident("x1"), "l")
)
```

Apparently we are dealing with the second of those two cases. Let's double check:

```scala
scala> q.ast match {
         case ReturningAction(Insert(entity: Entity, Nil), _, prop) => "first"
         case ReturningAction(action, alias, prop) => "second"
       }
res18: Boolean = second
```

## Changing `SqlIdiom`

Following the same approach as `ReturningClauseSupported`, but using `OUTPUT` instead of `RETURNING`, we have:

```scala
case r @ ReturningAction(action, alias, prop) =>
  idiomReturningCapability match {
    ...
    case ReturningClauseSupported =>
      stmt"${action.token} RETURNING ${returnListTokenizer.token(ExpandReturning(r)(this, strategy).map(_._1))}"
    case OutputClauseSupported =>
      stmt"${action.token} OUTPUT ${returnListTokenizer.token(ExpandReturning(r)(this, strategy).map(_._1))}"
```

That change generates:

```scala
scala> ctx.run(q).string
<console>:23: INSERT INTO TestEntity (s,i,l,o) VALUES (?, ?, ?, ?) OUTPUT l
```

Okay, `${action.token}` becomes `INSERT INTO TestEntity (s,i,l,o) VALUES (?, ?, ?, ?)`, so it's necessary to extract some information from `action` in order to generate the insert clause. Quill [already does that](https://github.com/getquill/quill/blob/3c97b8de95c9117f0da1f5cf892964f6c5431934/quill-sql/src/main/scala/io/getquill/context/sql/idiom/SqlIdiom.scala#L432):

```scala
// line 432
case Insert(entity: Entity, assignments) =>
  val table = insertEntityTokenizer.token(entity)
  val columns = assignments.map(_.property.token)
  val values = assignments.map(_.value)
  stmt"INSERT $table${actionAlias.map(alias => stmt" AS ${alias.token}").getOrElse(stmt"")} (${columns.mkStmt(",")}) VALUES (${values.map(scopedTokenizer(_)).mkStmt(", ")})"
```

Combining this code with our knowledge about the existing `RETURNING` clause to generate the `OUTPUT` clause, we have:

```scala
case OutputClauseSupported => action match {
  case Insert(entity: Entity, assignments) =>
    val table = insertEntityTokenizer.token(entity)
    val columns = assignments.map(_.property.token)
    val values = assignments.map(_.value)
    stmt"INSERT $table${actionAlias.map(alias => stmt" AS ${alias.token}").getOrElse(stmt"")} (${columns.mkStmt(",")}) OUTPUT ${returnListTokenizer.token(ExpandReturning(r)(this, strategy).map(_._1))} VALUES (${values.map(scopedTokenizer(_)).mkStmt(", ")})"
  case other =>
    fail(s"Action ast can't be translated to sql: '$other'")
}
```

Back to `sbt console`:

```scala
scala> ctx.run(q).string
<console>:20: INSERT INTO TestEntity (s,i,l,o) OUTPUT l VALUES (?, ?, ?, ?)
```

Almost there! We now have to find a way to include `INSERTED.` before every single element of `returnListTokenizer`. [`ExpandReturning`](https://github.com/getquill/quill/blob/f1c8eb45b86aaa691e1f4017e28457ef13e65504/quill-core/src/main/scala/io/getquill/norm/ExpandReturning.scala#L38) is the object handling that:

```scala
def apply(returning: ReturningAction)(idiom: Idiom, naming: NamingStrategy): List[(Ast, Statement)] = {
  val ReturningAction(_, alias, properties) = returning

  val dePropertized =
    Transform(properties) {
      case `alias` => ExternalIdent(alias.name)
    }
  ...
```

And I'll be entirely honest here, I have no idea what to do. I've never seen this part of the code.

## Asking for help

> When contributing to open source, never be afraid to ask for help

[And that's what I did](https://github.com/getquill/quill/pull/1681#discussion_r344310404). I actually [came up with a solution](https://github.com/getquill/quill/pull/1681#pullrequestreview-314385820) before [deusaquilus](https://twitter.com/deusaquilus)'s review, but it hadn't used the code already in place to solve that problem - and I knew that. Living and learning!

```scala
def apply(returning: ReturningAction, renameAlias: Option[String] = None)(idiom: Idiom, naming: NamingStrategy): List[(Ast, Statement)] = {
  val ReturningAction(_, alias, properties) = returning

  val dePropertized = renameAlias match {
    case Some(newName) =>
      BetaReduction(properties, alias -> Ident(newName))
    case None =>
      BetaReduction(properties, alias -> ExternalIdent(alias.name))
  }
  ...
```

My knowledge regarding `BetaReduction` is limited, so let's trust deusaquilus advice here. Let's make use of the new resource in `SqlIdiom`, but we should extract the duplicated code first:

```scala
case Insert(entity: Entity, assignments) =>
  val (table, columns, values) = insertInfo(insertEntityTokenizer, entity, assignments)
  stmt"INSERT $table${actionAlias.map(alias => stmt" AS ${alias.token}").getOrElse(stmt"")} (${columns.mkStmt(",")}) VALUES (${values.map(scopedTokenizer(_)).mkStmt(", ")})"
...

private def insertInfo(insertEntityTokenizer: Tokenizer[Entity], entity: Entity, assignments: List[Assignment])(implicit astTokenizer: Tokenizer[Ast]) = {
  val table = insertEntityTokenizer.token(entity)
  val columns = assignments.map(_.property.token)
  val values = assignments.map(_.value)
  (table, columns, values)
}
```

And finally, let's introduce `INSERTED`:
```scala
case OutputClauseSupported => action match {
  case Insert(entity: Entity, assignments) =>
    val (table, columns, values) = insertInfo(insertEntityTokenizer, entity, assignments)
    stmt"INSERT $table${actionAlias.map(alias => stmt" AS ${alias.token}").getOrElse(stmt"")} (${columns.mkStmt(",")}) OUTPUT ${returnListTokenizer.token(ExpandReturning(r, Some("INSERTED"))(this, strategy).map(_._1))} VALUES (${values.map(scopedTokenizer(_)).mkStmt(", ")})"
  case other =>
    fail(s"Action ast can't be translated to sql: '$other'")
}
```

Now, in the console we have:

```scala
scala> ctx.run(q).string
<console>:20: INSERT INTO TestEntity (s,i,l,o) OUTPUT INSERTED.l VALUES (?, ?, ?, ?)
```

And the tests are green again. Let's cover the other possibilities for `returning`:

```scala
"output clause - multi" in testContext.withDialect(MirrorSqlDialectWithOutputClause) { ctx =>
  import ctx._
  val q = quote {
    qr1.insert(lift(TestEntity("s", 0, 1L, None))).returning(r => (r.i, r.l))
  }
  val mirror = ctx.run(q)
  mirror.string mustEqual "INSERT INTO TestEntity (s,i,l,o) OUTPUT INSERTED.i, INSERTED.l VALUES (?, ?, ?, ?)"
  mirror.returningBehavior mustEqual ReturnRecord
}
"output clause - operation" in testContext.withDialect(MirrorSqlDialectWithOutputClause) { ctx =>
  import ctx._
  val q = quote { qr1.insert(lift(TestEntity("s", 0, 1L, None))).returning(r => (r.i, r.l + 1)) }
  val mirror = ctx.run(q)

  mirror.string mustEqual "INSERT INTO TestEntity (s,i,l,o) OUTPUT INSERTED.i, INSERTED.l + 1 VALUES (?, ?, ?, ?)"
}
"output clause - record" in testContext.withDialect(MirrorSqlDialectWithOutputClause) { ctx =>
  import ctx._
  val q = quote {
    qr1.insert(lift(TestEntity("s", 0, 1L, None))).returning(r => r)
  }
  val mirror = ctx.run(q)
  mirror.string mustEqual "INSERT INTO TestEntity (s,i,l,o) OUTPUT INSERTED.s, INSERTED.i, INSERTED.l, INSERTED.o VALUES (?, ?, ?, ?)"
  mirror.returningBehavior mustEqual ReturnRecord
}
"output clause - embedded" - {
  "embedded property" in testContext.withDialect(MirrorSqlDialectWithOutputClause) { ctx =>
    import ctx._
    val q = quote {
      qr1Emb.insert(lift(TestEntityEmb(Emb("s", 0), 1L, None))).returning(_.emb.i)
    }
    val mirror = ctx.run(q)
    mirror.string mustEqual "INSERT INTO TestEntity (s,i,l,o) OUTPUT INSERTED.i VALUES (?, ?, ?, ?)"
    mirror.returningBehavior mustEqual ReturnRecord
  }
  "two embedded properties" in testContext.withDialect(MirrorSqlDialectWithOutputClause) { ctx =>
    import ctx._
    val q = quote {
      qr1Emb.insert(lift(TestEntityEmb(Emb("s", 0), 1L, None))).returning(r => (r.emb.i, r.emb.s))
    }
    val mirror = ctx.run(q)
    mirror.string mustEqual "INSERT INTO TestEntity (s,i,l,o) OUTPUT INSERTED.i, INSERTED.s VALUES (?, ?, ?, ?)"
    mirror.returningBehavior mustEqual ReturnRecord
  }
}
```

Great, everything works! Now we have to handle a fundamental difference between `OUTPUT` and `RETURNING`.

## Stopping invalid actions

The next test is `"with returning clause - query"`, with this code:

```scala
val q = quote {
  qr1
    .insert(lift(TestEntity("s", 0, 1L, None)))
    .returning(r => (query[Dummy].map(d => d.i).max))
}
```

Unlike Postgres, [SQL Server doesn't allow an output clause in a select statement](https://stackoverflow.com/a/48932734/973740), meaning that our implementation has to fail the compilation when a query is present. Let's start with a test:

```scala
"output clause - should fail on query" in testContext.withDialect(MirrorSqlDialectWithOutputClause) { ctx =>
  """import ctx._; quote { qr4.insert(lift(TestEntity4(1L))).returning(r => query[TestEntity4].filter(t => t.i == r.i)) }""" mustNot compile
}
```

Recapping the beginning of our session:

> `Parsing` verifies the type of the returning clause and can fail the compilation if necessary

That is exactly what we need:

```scala
// line 831
def verifyAst(returnBody: Ast) = capability match {
  case OutputClauseSupported =>
  case ReturningClauseSupported =>
  // Only .returning(r => r.prop) or .returning(r => OneElementCaseClass(r.prop1..., propN)) or .returning(r => (r.prop1..., propN)) (well actually it's prop22) is allowed.
  case ReturningMultipleFieldSupported =>
    returnBody match {
  ...
```

In order to figure the right moment to stop the compilation, we need to understand that AST better. Back to the console:

```scala
val q = quote { qr4.insert(lift(TestEntity4(1L))).returning(r => query[TestEntity4].filter(t => t.i == r.i)) }

scala> pprint.pprintln(q.ast, 200)
Returning(
  Insert(
    Entity("TestEntity4", List()),
    List(Assignment(Ident("v"), Property(Ident("v"), "i"), ScalarValueLift("$line8.$read.$iw.$iw.$iw.$iw.$iw.$iw.TestEntity4.apply(1L).i", 1L, MirrorEncoder(<function3>))))
  ),
  Ident("r"),
  Filter(Entity("TestEntity4", List()), Ident("t"), BinaryOperation(Property(Ident("t"), "i"), ==, Property(Ident("r"), "i")))
)
```

The last line is the `returnBody` we are looking for. Having `.map` instead of `.filter` it generates:

```scala
val q2 = quote { qr4.insert(lift(TestEntity4(1L))).returning(r => query[TestEntity4]).map(t => t.i) }

scala> pprint.pprintln(q2.ast, 200)
Returning(
  ...
  Map(Entity("TestEntity4", List()), Ident("x1"), Property(Ident("x1"), "i"))
)
```

None of these can be allowed to compile. Many similar operations extend [`Query`](https://github.com/getquill/quill/blob/master/quill-core/src/main/scala/io/getquill/ast/Ast.scala#L35), so let's consider that to define a limitation in `Parsing`:

```scala
implicit class InsertReturnCapabilityExtension(capability: ReturningCapability) {
  def verifyAst(returnBody: Ast) = capability match {
    case OutputClauseSupported =>
      returnBody match {
        case _: Query =>
          c.fail(s"${currentIdiom.map(n => s"The dialect $n does").getOrElse("Unspecified dialects do")} not allow queries in 'returning' clauses.")
        case _ =>
      }
    case ReturningClauseSupported =>
    ...
```

After this change, the tests are happy and green again. [We need the same tests for `.returningGenerated`](https://github.com/getquill/quill/blob/98de8aaef0c8611452b9434d40bc685a8a1cb71f/quill-sql/src/test/scala/io/getquill/context/sql/SqlActionMacroSpec.scala#L419), but the good news are that we need a quite similar change, which is trivial at this point:

```scala
// line 450
case r @ ReturningAction(Insert(table: Entity, Nil), alias, prop) =>
  idiomReturningCapability match {
    ...
    case ReturningClauseSupported =>
      stmt"INSERT INTO ${table.token} ${defaultAutoGeneratedToken(prop.token)} RETURNING ${returnListTokenizer.token(ExpandReturning(r)(this, strategy).map(_._1))}"
    case OutputClauseSupported =>
      stmt"INSERT INTO ${table.token} OUTPUT ${returnListTokenizer.token(ExpandReturning(r, Some("INSERTED"))(this, strategy).map(_._1))} ${defaultAutoGeneratedToken(prop.token)}"
    ...
```

It concludes the changes around the sql idiom.

## Changing `SQLServerDialect`

The dialect will extend `CanOutputClause` from now on:

```scala
trait SQLServerDialect
  extends SqlIdiom
  with QuestionMarkBindVariables
  with ConcatSupport
  with CanOutputClause {
```

We have to ensure that the SQL generated is valid against the database. Let's add some tests to [`JdbcContextSpec`](https://github.com/getquill/quill/blob/3232763ec5df444382aefcb6c87a6ff2b97fb237/quill-jdbc/src/test/scala/io/getquill/context/jdbc/sqlserver/JdbcContextSpec.scala):

```scala
"Insert with returning with multiple columns" in {
  testContext.run(qr1.delete)
  val inserted = testContext.run {
    qr1.insert(lift(TestEntity("foo", 1, 18L, Some(123)))).returning(r => (r.i, r.s, r.o))
  }
  (1, "foo", Some(123)) mustBe inserted
}

"Insert with returning with multiple columns and operations" in {
  testContext.run(qr1.delete)
  val inserted = testContext.run {
    qr1.insert(lift(TestEntity("foo", 1, 18L, Some(123)))).returning(r => (r.i + 100, r.s, r.o.map(_ + 100)))
  }
  (1 + 100, "foo", Some(123 + 100)) mustBe inserted
}

"Insert with returning with multiple columns and query embedded" in {
  testContext.run(qr1Emb.delete)
  testContext.run(qr1Emb.insert(lift(TestEntityEmb(Emb("one", 1), 18L, Some(123)))))
  val inserted = testContext.run {
    qr1Emb.insert(lift(TestEntityEmb(Emb("two", 2), 18L, Some(123)))).returning(r =>
      (r.emb.i, r.o))
  }
  (2, Some(123)) mustBe inserted
}

"Insert with returning with multiple columns - case class" in {
  case class Return(id: Int, str: String, opt: Option[Int])
  testContext.run(qr1.delete)
  val inserted = testContext.run {
    qr1.insert(lift(TestEntity("foo", 1, 18L, Some(123)))).returning(r => Return(r.i, r.s, r.o))
  }
  Return(1, "foo", Some(123)) mustBe inserted
}
```

What can go wrong?

```diff
[info] - Insert with returning with multiple columns *** FAILED ***
[info]   com.microsoft.sqlserver.jdbc.SQLServerException: A result set was generated for update.
[info]   at com.microsoft.sqlserver.jdbc.SQLServerException.makeFromDriverError(SQLServerException.java:227)
[info]   at com.microsoft.sqlserver.jdbc.SQLServerPreparedStatement.doExecutePreparedStatement(SQLServerPreparedStatement.java:592)
[info]   at com.microsoft.sqlserver.jdbc.SQLServerPreparedStatement$PrepStmtExecCmd.doExecute(SQLServerPreparedStatement.java:508)
[info]   at com.microsoft.sqlserver.jdbc.TDSCommand.execute(IOBuffer.java:7233)
[info]   at com.microsoft.sqlserver.jdbc.SQLServerConnection.executeCommand(SQLServerConnection.java:2869)
[info]   at com.microsoft.sqlserver.jdbc.SQLServerStatement.executeCommand(SQLServerStatement.java:243)
[info]   at com.microsoft.sqlserver.jdbc.SQLServerStatement.executeStatement(SQLServerStatement.java:218)
[info]   at com.microsoft.sqlserver.jdbc.SQLServerPreparedStatement.executeUpdate(SQLServerPreparedStatement.java:461)
[info]   at com.zaxxer.hikari.pool.ProxyPreparedStatement.executeUpdate(ProxyPreparedStatement.java:61)
[info]   at com.zaxxer.hikari.pool.HikariProxyPreparedStatement.executeUpdate(HikariProxyPreparedStatement.java)
```

This is an interesting error, which actually happens in [ProductJdbcSpec](https://github.com/getquill/quill/blob/f1c8eb45b86aaa691e1f4017e28457ef13e65504/quill-jdbc/src/test/scala/io/getquill/context/jdbc/sqlserver/ProductJdbcSpec.scala) as well. Luckly, [it has been reported in the issue](https://github.com/getquill/quill/issues/1497#issue-461855389):

> Also note that SQL Server requires `prep.executeQuery()` instead of a combination of `preparedStatement.executeUpdate()` and `preparedStatement.getGeneratedKeys()`

Good to know. [`JdbcContextBase`](https://github.com/getquill/quill/blob/6a50fff2168ee9cf00309b432cd6bb27a20a5cea/quill-jdbc/src/main/scala/io/getquill/context/jdbc/JdbcContextBase.scala) is the responsible for handling jdbc connections. According to the description above, the method we are looking for is `executeActionReturning`:

```scala
def executeActionReturning[O](sql: String, prepare: Prepare = identityPrepare, extractor: Extractor[O], returningBehavior: ReturnAction): Result[O] =
  withConnectionWrapped { conn =>
    val (params, ps) = prepare(prepareWithReturning(sql, conn, returningBehavior))
    logger.logQuery(sql, params)
    ps.executeUpdate()
    handleSingleResult(extractResult(ps.getGeneratedKeys, extractor))
  }
```

[`SqlServerJdbcContextBase`](https://github.com/getquill/quill/blob/8372054196410b22d193d1f49008e0f914161b83/quill-jdbc/src/main/scala/io/getquill/context/jdbc/BaseContexts.scala#L44) extends `JdbcContextBase`, defining `SQLServerDialect` as `idiom`:

```scala
trait SqlServerJdbcContextBase[N <: NamingStrategy] extends JdbcContextBase[SQLServerDialect, N]
  with BooleanObjectEncoding
  with UUIDStringEncoding {

  val idiom = SQLServerDialect
}
```

SQL Server demands an exceptional behaviour regarding `preparedStatement`, so we need to override `executeActionReturning`:

```scala
trait SqlServerJdbcContextBase[N <: NamingStrategy] extends JdbcContextBase[SQLServerDialect, N]
  with BooleanObjectEncoding
  with UUIDStringEncoding {

  val idiom = SQLServerDialect

  override def executeActionReturning[O](sql: String, prepare: Prepare = identityPrepare, extractor: Extractor[O], returningBehavior: ReturnAction): Result[O] =
    withConnectionWrapped { conn =>
      val (params, ps) = prepare(prepareWithReturning(sql, conn, returningBehavior))
      logger.logQuery(sql, params)
      handleSingleResult(extractResult(ps.executeQuery, extractor))
    }
}
```

Obviously we have to [add more tests to SQLServerDialectSpec](https://github.com/getquill/quill/blob/98de8aaef0c8611452b9434d40bc685a8a1cb71f/quill-sql/src/test/scala/io/getquill/context/sql/idiom/SQLServerDialectSpec.scala#L73), but this post is already long enough to paste those tests here. With that we finish our contribution to Quill.

That was fun, maybe you should be the pilot next time! Let me know when we should have the next session in the comments!
