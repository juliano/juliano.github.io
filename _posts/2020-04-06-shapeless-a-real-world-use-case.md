---
layout: single
title: "Shapeless: a real world use case"
header:
  teaser: /assets/images/water.jpg
  imgcredit: Image by PublicDomainPictures from Pixabay
categories:
  - scala
  - shapeless
---

[Shapeless](https://github.com/milessabin/shapeless) is a library for generic programming in Scala, largely [present in the ecosystem](https://github.com/milessabin/shapeless/wiki/Built-with-shapeless), but mostly behind the scenes; it is likely shapeless powers some of the libraries in your project, even though you don't use it directly.

Trying to solve an everyday problem, I've found an use case which I could solve using shapeless. This post doesn't intend to explain how shapeless work (there is a [whole book about it here](https://underscore.io/books/shapeless-guide/)), but to provide a taste of it instead.

## The Challenge

We need to write a REST API consumer using [ZIO](https://zio.dev/) and [http4s](https://http4s.org/). Here is the service definition:

```scala
import io.circe.Decoder
import zio.{Has, RIO, Task}

object HttpClient {
  type HttpClient = Has[Service]

  trait Service {
    protected final val rootUrl = "http://localhost:8080"

    def get[T](uri: String, parameters: Map[String, String])
              (implicit d: Decoder[T]): Task[List[T]]
  }

  def get[T](resource: String, parameters: Map[String, String])
            (implicit d: Decoder[T]): RIO[HttpClient, List[T]] =
    RIO.accessM[HttpClient](_.get.get[T](resource, parameters))
}
```

In case you are not familiar with ZIO (you should, it's awesome), what you need to know is:

- every get requests returns a `Task` of `List`
- the `get` function outside the `Service` is just a helper to access the environment of the effect (ZIO stuff)

[Learn more about ZIO modules and layers here](https://zio.dev/docs/howto/howto_use_layers).

This is the `HttpClient.Service` implementation, using [http4s](https://http4s.org/v0.21/client/):

```scala
import io.circe.Decoder
import org.http4s.Uri
import org.http4s.circe.CirceEntityCodec.circeEntityDecoder
import org.http4s.client.Client
import org.http4s.client.dsl.Http4sClientDsl
import zio._
import zio.interop.catz._

class Http4sClient(client: Client[Task])
  extends HttpClient.Service with Http4sClientDsl[Task] {

  def get[T](resource: String, parameters: Map[String, String])
            (implicit d: Decoder[T]): Task[List[T]] = {
    val uri = Uri(path = rootUrl + resource)
      .withQueryParams(parameters)

    client
      .expect[List[T]](uri.toString())
      .foldM(IO.fail(_), ZIO.succeed(_))
  }
}
```

`Http4sClient.get` adds `resource` to the `uri` and the `parameters` are the query string. Now, to represent the request call, we have a `case class` called `OrganisationRequest`:

```scala
case class OrganisationRequest(code: Option[String],
                               description: Option[String],
                               page: Integer = 1)
```

## The Problem

Using the client (via `get` helper) is trivial, except for one detail:

```scala
import HttpClient.get

def organisations(request: OrganisationRequest):
  get[Organisation]("/organisations", ???)
```

We need to transform the `request` into `Map[String, String]`, what is an easy task. However, there are many "request" objects, and writing `toMap` methods to every single one of them is a Java-ish solution. Here is the challenge: how can we build this generic transformation?

Spoiler: with shapeless.

## A bit of shapeless

This section is a grasp of how shapeless works, so the solution will make more sense when we get there. Shapeless can create an [Heterogenous List](https://github.com/milessabin/shapeless/wiki/Feature-overview:-shapeless-2.0.0#heterogenous-lists) (or `HList`) as a generic representation of case classes. Let's do it using [`Generic`](https://github.com/milessabin/shapeless/blob/master/core/src/main/scala/shapeless/generic.scala#L102):

```scala
scala> import shapeless._

scala> val org = OrganisationRequest(Some("acme"), None, 5)
org: OrganisationRequest = OrganisationRequest(Some(org),None,5)

scala> val gen = Generic[OrganisationRequest]
gen: shapeless.Generic[OrganisationRequest]
  {type Repr =
    Option[String]
    :: Option[String]
    :: Integer
    :: shapeless.HNil} = anon$macro$4$1@48f146f2

scala> gen.to(org)
res8: gen.Repr = Some(acme) :: None :: 5 :: HNil
```

The generic representation of `OrganisationRequest` is an `HList` of type `Option[String] :: Option[String] :: Int :: HNil`. We have the values, but we need the names of the fields for our `Map`. We need `LabelledGeneric` instead of `Generic`:

```scala
scala> val lgen = LabelledGeneric[OrganisationRequest]
lgen: shapeless.LabelledGeneric[OrganisationRequest]
  {type Repr =
    Option[String] with shapeless.labelled.KeyTag[Symbol with shapeless.tag.Tagged[String("code")],Option[String]]
    :: Option[String] with shapeless.labelled.KeyTag[Symbol with shapeless.tag.Tagged[String("description")],Option[String]]
    :: Integer with shapeless.labelled.KeyTag[Symbol with shapeless.tag.Tagged[String("page")],Integer]
    :: shapeless.HNil} = shapeless.LabelledGeneric$$anon$1@55f78c67
```

As you can see, with `LabelledGeneric` it's possible to retain the information about the field names as well.

## The Solution

Luckily, we don't need to manipulate `LabelledGeneric` ourselves, shapeless provides us with plenty of useful type classes that can be found in the `shapless.ops` package. We will build our solution using [`ToMap`](https://github.com/milessabin/shapeless/blob/master/core/src/main/scala/shapeless/ops/products.scala#L90):

```scala
scala> import shapeless.ops.product.ToMap

scala> val toMap = ToMap[OrganisationRequest]
toMap: shapeless.ops.product.ToMap[OrganisationRequest]
  {type K = Symbol
    with shapeless.tag.Tagged[_ >: String("page")
    with String("description")
    with String("code") <: String];
  type V = java.io.Serializable} =
    shapeless.ops.product$ToMap$$anon$5@3bccd311

scala> val map = toMap(org)
map: toMap.Out = Map('page -> 5,
                     'description -> None,
                     'code -> Some(acme))
```

We can make it even nicer using shapeless syntax:

```scala
scala> import shapeless.syntax.std.product._

scala> val map = org.toMap[Symbol, Any]
map: Map[Symbol,Any] = Map('page -> 5,
                           'description -> None,
                           'code -> Some(acme))
```

For the final solution, let's create an `implicit class` in order to add a `parameters` method to our `request` class. Besides, we should remove every entry with `null` or `None` values, flatten the `Option`s and turn keys and values into `String`:

```scala
import shapeless.ops.product.ToMap
import shapeless.syntax.std.product._

implicit class RequestOps[A <: Product](val a: A) {
  def parameters(implicit toMap: ToMap.Aux[A, Symbol, Any]): Map[String, String] =
    a.toMap[Symbol, Any]
      .filter {
        case (_, v: Option[Any]) => v.isDefined
        case (_, v) => v != null
      }
      .map {
        case (k, v: Option[Any]) => k.name -> v.get.toString
        case (k, v) => k.name -> v.toString
      }
}
```

A few comments here:

- `A <: Product` needs to be in place so we can use `shapeless.ops.product`. [All case classes implement Product](https://www.scala-lang.org/api/2.13.1/scala/Product.html), it's just a matter of adding the constrain for implicit resolution;
- the implicit parameter `toMap` is a `ToMap.Aux` instead of just `ToMap`. Long story short, shapeless defines the `Aux` alias in order to make some of its internal complexity more readable and usable. Just trust me here ;)

Finally, this brings us to an elegant solution:

```scala
import HttpClient.get
import RequestOps

def organisations(request: OrganisationRequest):
  get[Organisation]("/organisations", request.parameters)
```

## Conclusion

Even though shapeless looks almost magical at the first glance, after dedicating myself to understand it better, I've figured it can be very useful in practical terms. Shapeless provides a broad range of typeclasses that can be used in all sort of ways, and spending time learning about how they work is a very interesting exercise, improving skills related to typeclasses, derivations and bringing clarity about how some popular libraries that use shapeless work, like [circe](https://circe.github.io/circe/).

I've heard that adding too much shapeless can properly affect the project's compile time. I'd like to hear more about it, if you have experience using shapeless directly, please share in the comments.
