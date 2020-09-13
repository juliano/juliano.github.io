---
layout: single
title: "Quill: Translating Boolean Literals"
header:
  teaser: /assets/images/egypt-inscriptions.jpg
  imgcredit: Image by photosforyou from Pixabay
categories:
  - scala
  - quill
  - opensource
  - howto
classes: wide
---

Quill is an wonderful tool, and like most wonderful tools it solves very complex problems - but not all of them. In this post you will learn about some of the challenges the ones on the backstage have been dealing with while developing Quill, and the interesting solutions that emerged to beat those challenges.

## Houston, we have a problem!

Subquery expansion, aliasing/renaming clauses or handling booleans in SQL databases which do not support `true`/`false` values are complex features that ended up producing even more complex solutions in the codebase overtime. Let's stick to the "handling booleans" issue to illustrate the problem. Consider the following `quote`:

```scala
case class TestEntity(id: Long)
val q = quote {
  query[TestEntity].map(t => if (true) true else false)
}
```

Running the quote with `PostgresJdbcContext` (or any other context which have the type boolean) we would have the following output:

```sql
SELECT CASE WHEN true THEN true ELSE false END FROM TestEntity t
```

But running the same quote in a context where `true` and `false` don't exist (SQL Server or Oracle), the output is:

```sql
SELECT CASE WHEN 1=1 THEN 1=1 ELSE 1=0 END FROM TestEntity t
```

Which is incorrect, it should be:

```sql
SELECT CASE WHEN 1=1 THEN 1 ELSE 0 END FROM TestEntity t
```

The challenge here is to figure out whether a boolean should be translated into an expression (`1=1`) or into a value (the `1` and `0` after `THEN/ELSE`). Doing so would be a nightmare; given the multitude of possibilities, the decision would be made through assumptions at best and guesses at worst. In reality, the issue described is only a symptom; just the tip of the iceberg.

The oversimplified version of the real problem is: Quill AST elements don't have enough information about what they are representing: an alias doesn't know which fields it actually has, a boolean doesn't know if it is a value or an expression. If you want to understand it properly, [here you can see how deep the rabbit hole goes](https://github.com/getquill/quill/pull/1911){:target="_blank"}.

The strategy to solve this problem? Including more information in the AST.

## Quill-Application-Types (*Quats*)

Borrowing [Alex](https://github.com/deusaquilus){:target="_blank"}'s explanation from the link above:

> Every AST node will have `a.quat` field that represents it's Quill-Application-Type. Since most AST elements merely reflect their inner types, only things like `Ident`, `Infix`, and several others actually need to have a Quat-type directly inferenced from the parser.

The rabbit whole link is the actual pull request adding Quats to the project, a remarkable work by Quill's lead developer (thanks again, Alex!). It introduces four Quats:

* **Quat.Product**, representing Case Classes and Tuples
* **Quat.Value**, representing values
* **Quat.Null** and **Quat.Generic**, representing types whose values are not fully known yet.

Now we will introduce *Quats* to represent boolean values and expressions. A [complete description of the problem is in the issue here](https://github.com/getquill/quill/issues/1685#issuecomment-662275696){:target="_blank"}, let's do it! :grin:

## Introducing new Quats: `BooleanValue` and `BooleanExpression`

The new *Quats* will sit in the same level as `Value`. In [Quat.scala](https://github.com/getquill/quill/blob/master/quill-core-portable/src/main/scala/io/getquill/quat/Quat.scala){:target="_blank"}:

```scala
// line 225
case object Value extends Quat
case object BooleanValue extends Quat
case object BooleanExpression extends Quat
```

The next step is to add boolean quats to the `Ast`. How do we know which element could use the new *Quats*? In the first moment we can't fully tell, besides, this is irrelevant until the translation of the `Ast` happens. In other words, **we can change the obvious elements in the very beginning and transform everything else later**. Seems complicated? Baby steps then!

### Obvious cases

The so called "obvious cases" are the situations where `Ast` elements can have their `quats` defined during their creation, when Quill reads the code inside the `quote` and build the Ast. `Constant` is the first one, which has `quat` fixed as `Value` at the moment:

```scala
// Ast.scala
case class Constant(v: Any) extends Value { def quat = Quat.Value }
```

Its `quat` has to be defined according to the received value, being a `Boolean` the `quat` will be a `BooleanValue`, otherwise, `Value`:

```scala
// line 434
case class Constant(v: Any, quat: Quat) extends Value

object Constant {
  def apply(v: Any) = {
    val quat = if (v.isInstanceOf[Boolean]) Quat.BooleanValue else Quat.Value
    new Constant(v, quat)
  }
}
```

The same change happens to `Dynamic`, [as you can see here](https://github.com/getquill/quill/blob/master/quill-core-portable/src/main/scala/io/getquill/ast/Ast.scala#L526){:target="_blank"}. Those are all the cases where we can apply `BooleanValue` at this stage.

### Not so obvious cases

Initially, an obvious case for `BooleanExpression` is `filter`. This is the `Filter` ast definition:

```scala
case class Filter(query: Ast, alias: Ident, body: Ast) extends Query { def quat = query.quat }
```

`Ast` elements are composed by other `Ast` elements. The `body` inside a `filter` is by definition an expression, so `body.quat` needs to be a `BooleanExpression`, even in a quote like:

```scala
case class MyEntity(s: String, i: Int)
val q = quote {
  query[MyEntity].filter(t => true)
}
```

However, a deeper look into `q.ast` will reveal a tricky situation:

```scala
pprint.pprintln(q2.ast, 200)
Filter(
  Entity("MyEntity", List(), Product(Map("s" -> V, "i" -> V))),
  Ident("t", Product(Map("s" -> V, "i" -> V))),
  Constant(true, BV)
)
```

`Constant(true, BV)` is the body of `Filter`, has a `quat` which is a `BooleanValue` as defined earlier. Even though we need a `BooleanExpression`, the universe of possibilities for `body` would demand it to be reduced first before deciding for the correct `quat`. That will happen later, so we can leave it for now.

Actually, most of the heavy lifting should happen during the AST transformation phase for one main reason:

> `Boolean Quats` are relevant only for dialects that don't support booleans, all the others can ignore this information.

That said, there are two more elements which we can already change their `quats`. The `quat` of an `UnaryOperation` will always be a `BooleanExpression`:

```scala
// line 409
case class UnaryOperation(operator: UnaryOperator, ast: Ast) extends Operation {
  def quat = Quat.BooleanExpression
}
```

Any `BinaryOperation` with an `operator` that produces a boolean outcome will be have a `BooleanExpression` in its `quat` as well:

```scala
// line 411
case class BinaryOperation(a: Ast, operator: BinaryOperator, b: Ast) extends Operation {
  import BooleanOperator._
  import NumericOperator._
  import StringOperator.`startsWith`
  import SetOperator.`contains`

  def quat = operator match {
    case EqualityOperator.`==` | EqualityOperator.`!=`
      | `&&` | `||`
      | `>` | `>=` | `<` | `<=`
      | `startsWith`
      | `contains` =>
      Quat.BooleanExpression
    case _ =>
      Quat.Value
  }
}
```

The easy part is done. The real challenge starts now!

## AST transformation phase

The new phase will be called `VendorizeBooleans`. It will extend `StatelessTransformer` and perform transformations when [`Idiom.translate`](https://github.com/getquill/quill/blob/master/quill-core-portable/src/main/scala/io/getquill/idiom/Idiom.scala#L15){:target="_blank"} is called. This is the moment where we need to define if our booleans are values or expressions. The rules are:

**For any slot requiring a `BooleanExpression`:**
  - if the `quat` is a `BooleanValue bv`, convert it to `bv = true` which will correctly be a `BooleanExpression`, and later become `bv = 1`;
  - if the `quat` is anything else, we are done

**For any slot requiring a `BooleanValue`:**
  - if the `quat` is a `BooleanExpression be`, convert it to `if (be) true else false` which will correctly be a `BooleanValue`, and later become `1` or `0`, after the `if` is reduced;
  - again, if the `quat` is anything else, we are done

Remember, the transformation happens in the AST level. The names `expressifyValue` and `valuefyExpression` sound appropriate:

```scala
package io.getquill.sql.norm

import io.getquill.ast.Implicits.AstOpsExt
import io.getquill.ast._
import io.getquill.quat.Quat.{ BooleanExpression, BooleanValue }

object VendorizeBooleans extends StatelessTransformer {

  def expressifyValue(ast: Ast): Ast = ast.quat match {
    case BooleanValue => Constant(true, BooleanValue) +==+ ast
    case _            => ast
  }

  def valuefyExpression(ast: Ast): Ast = ast.quat match {
    case BooleanExpression => If(ast, Constant(true, BooleanValue), Constant(false, BooleanValue))
    case _                 => ast
  }
}
```

After spending a (loooong) time testing a breaking things, I figured out the elements which need their booleans to be vendorized:

- the `body` of `Filter` needs to be an **expression**;
- the `body` of `If` needs to be an **expression** but `then` and `else` need to be a **value**;
- both `Join` and `FlatJoin` need `on` to be an **expression**;
- anything else can be left to the SQL interpreter (`super.apply`)

In the code:

```scala
override def apply(ast: Ast): Ast =
  ast match {
    case Filter(q, alias, body) =>
      Filter(apply(q), alias, expressifyValue(apply(body)))
    case If(cond, t, e) =>
      If(expressifyValue(apply(cond)), valuefyExpression(apply(t)), valuefyExpression(apply(e)))
    case Join(typ, a, b, aliasA, aliasB, on: Constant) =>
      Join(typ, apply(a), apply(b), aliasA, aliasB, expressifyValue(apply(on: Ast)))
    case FlatJoin(typ, a, aliasA, on) =>
      FlatJoin(typ, a, aliasA, expressifyValue(apply(on)))
    case _ =>
      super.apply(ast)
  }
```

So far so good. However, `Ast` elements are composed by other `Ast` elements, remember? That means the logic needs to be applied recursively. Let's turn our attention to the ones being expressified; those elements will be mostly `BinaryOperation`s or `UnaryOperation`s.

#### Transforming `Operations`

A quick reminder about how `BinaryOperation`s and `UnaryOperation`s look like:

```scala
case class UnaryOperation(operator: UnaryOperator, ast: Ast) extends Operation
case class BinaryOperation(a: Ast, operator: BinaryOperator, b: Ast) extends Operation
```

The `operator` gives us the informations we need to define how the transformations need to happen. Building few `objects` can be very handy to express how they can be identified.

##### `OperatorOnExpressions`

The operators `||` and `&&` are typically present in expressions: `true || e.isSomething` will be converted to `true == true || e.isSomething == true` and then tokenized as `1 == 1 || e.isSomething == 1`.

```scala
object OperatorOnExpressions {
  import BooleanOperator._

  def unapply(op: BinaryOperator) = op match {
      case `||` | `&&` => Some(op)
      case _           => None
    }
}
```

##### `OperatorOnValues`

The operators `==`, `!=`, `<`, `>`, `<=`, `>=` compare values: `true == e.isSomething` will be converted to `if (true == e.isSomething) true else false` and tokenized to `if (1 == e.isSomething) 1 else 0`.

```scala
object OperatorOnValues {
  import NumericOperator._

  def unapply(op: BinaryOperator) = op match {
      case `<` | `>` | `<=` | `>=` | EqualityOperator.`==` | EqualityOperator.`!=` => Some(op)
      case _ => None
    }
}
```

##### `StringTransformerOperation`

Operations transforming strings are an exception to the role above and can't be valuefied.

```scala
object StringTransformerOperation {
    import StringOperator._

    def unapply(op: UnaryOperation) = op.operator match {
        case `toUpperCase` | `toLowerCase` | `toLong` | `toInt` => Some(op)
        case _ => None
      }
  }
```

##### A new transformation arises!

Those rules applied will complete [`VendorizeBooleans`](https://github.com/getquill/quill/blob/master/quill-sql-portable/src/main/scala/io/getquill/sql/norm/VendorizeBooleans.scala){:target="_blank"}:

```scala
override def apply(operation: Operation): Operation = {
  import BooleanOperator._

  operation match {
    case BinaryOperation(a, OperatorOnExpressions(op), b) =>
      BinaryOperation(expressifyValue(apply(a)), op, expressifyValue(apply(b)))

    case BinaryOperation(a, OperatorOnValues(op), b) => {
      (a, b) match {
        case (StringTransformerOperation(_), StringTransformerOperation(_)) =>
          BinaryOperation(apply(a), op, apply(b))
        case (StringTransformerOperation(_), _) =>
          BinaryOperation(apply(a), op, valuefyExpression(apply(b)))
        case (_, StringTransformerOperation(_)) =>
          BinaryOperation(valuefyExpression(apply(a)), op, apply(b))
        case _ =>
          BinaryOperation(valuefyExpression(apply(a)), op, valuefyExpression(apply(b)))
      }
    }

    // similar to the first case
    case UnaryOperation(`!`, ast) =>
      UnaryOperation(BooleanOperator.`!`, expressifyValue(apply(ast)))

    case _ =>
      super.apply(operation)
  }
}
```

Now we can just use it!

### Translating phase

As mentioned before, `VendorizeBooleans` should happen when `Idiom.translate` is called, or being more specific, when `SqlIdiom.translate` is called, given the feature is relevant only for sql databases.

We will introduce a special version of `SqlIdiom`, which overrides `translate` and `valueTokenizer` - the guy who generates `0`s and `1`s. It will be called `BooleanLiteralSupport`:

```scala
package io.getquill.sql.idiom

import ...

trait BooleanLiteralSupport extends SqlIdiom {

  override def translate(ast: Ast)(implicit naming: NamingStrategy): (Ast, Statement) = {
    val normalizedAst = VendorizeBooleans(SqlNormalize(ast))
    implicit val tokernizer = defaultTokenizer

    val token =
      normalizedAst match {
        case q: Query =>
          val sql = querifyAst(q)
          trace("sql")(sql)
          VerifySqlQuery(sql).map(fail)
          val expanded = ExpandNestedQueries(sql)
          trace("expanded sql")(expanded)
          val refined = if (Messages.pruneColumns) RemoveUnusedSelects(expanded) else expanded
          trace("filtered sql (only used selects)")(refined)
          val cleaned = if (!Messages.alwaysAlias) RemoveExtraAlias(naming)(refined) else refined
          trace("cleaned sql")(cleaned)
          val tokenized = cleaned.token
          trace("tokenized sql")(tokenized)
          tokenized
        case other =>
          other.token
      }

    (normalizedAst, stmt"$token")
  }

  override implicit def valueTokenizer(implicit astTokenizer: Tokenizer[Ast], strategy: NamingStrategy): Tokenizer[Value] =
    Tokenizer[Value] {
      case Constant(b: Boolean, Quat.BooleanValue) =>
        StringToken(if (b) "1" else "0")
      case Constant(b: Boolean, Quat.BooleanExpression) =>
        StringToken(if (b) "1 = 1" else "1 = 0")
      case other =>
        super.valueTokenizer.token(other)
    }
}
```

And we are finally done. Have a look at [`BooleanLiteralSupportSpec`](https://github.com/getquill/quill/blob/master/quill-sql/src/test/scala/io/getquill/context/sql/idiom/BooleanLiteralSupportSpec.scala){:target="_blank"}, where you can see some examples covered by the awesome new feature!

Last but not least, let's make use of that bad boy. Sql Server and Oracle dialects are waiting for it!

```scala
trait SQLServerDialect extends SqlIdiom
  ...
  with BooleanLiteralSupport {
  // remove valueTokenizer
}

trait OracleDialect extends SqlIdiom
  ...
  with BooleanLiteralSupport {
}
```

## Conclusion

Phew, that was a long ride! We covered a lot in this post:

- A complex problem sitting at the insides of Quill
- The solution brought by Quats
- Introducing new Quats
- Going through a whole new transformation phase
- Translating booleans and adapting dialects

I hope you've enjoyed the complexities and challenges! [The complete pull request is here!](https://github.com/getquill/quill/pull/1923){:target="_blank"}
