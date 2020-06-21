---
layout: single
title: "ZIO + Http4s: a simple API client"
header:
  teaser: /assets/images/msg-bottle.jpg
  imgcredit: Image by Comfreak from Pixabay
categories:
  - scala
  - zio
  - http4s
---

Discussing with a brazilian friend about the situation in our country, we realised how difficult it is to find information about public spending, and when available, how difficult it can be to reason about it. Joining our forces, we decided to explore some data exposed by the Brazilian government, aiming to provide an easier way to visualise and understand how the public resources has been used.

The starting point would be: finding some data to analyse, that is relatively easy (at least from a developer's perspective) to collect. A good candidate is the [*Portal da Transparência*](http://portaltransparencia.gov.br/api-de-dados){:target="_blank"} (in literal translation, Transparency Portal), an initiative to make public data available via APIs or downloading CSV files.

Is there a better way to learn about an API than writing a client for it? So let's do it with [ZIO](https://zio.dev) + [http4s client](https://http4s.org/v0.21/client/){:target="_blank"}!

<a href="https://zio.dev/">
![](/assets/images/zio-logo.jpg){: .align-center}
</a>

## Why ZIO?

After [my talk in Scala UA](https://www.scalaua.com/agenda/){:target="_blank"}, someone asked me what has called my attention in the Scala ecossystem recently. I believe ZIO can be a game changer, because it is not "just for functional programmers". Even though it is strongly based in functional principles, it doesn't assume the users already understand functional concepts (this is just a *Monad*!), which can be scary for new joiners.

Among all the powerful features ZIO provides, it's designed to be easy to use and adopt, what is from my perspective, by far, its best feature. [#ScalaThankYou](https://twitter.com/hashtag/ScalaThankYou){:target="_blank"} ZIO Team!

Time to code, let's start [defining a ZIO module](https://zio.dev/docs/howto/howto_use_layers#our-first-zio-module){:target="_blank"}.

## The `HttpClient` module

The API supports only `GET` requests, what makes the trait definition very simple:

```scala
package pdt.http

import io.circe.Decoder
import org.http4s.client.Client
import zio._

object HttpClient {
  type HttpClient = Has[Service]

  trait Service {
    protected val rootUrl = "http://www.transparencia.gov.br/api-de-dados/"

    def get[T](uri: String, parameters: Map[String, String])
              (implicit d: Decoder[T]): Task[T]
  }

  def http4s: ZLayer[Has[Client[Task]], Nothing, HttpClient] = ???
}
```

`Service` has only one method `get[T]` with arguments `resource: String` and `parameters: Map[String, String]`, which will become part of the url in the format `"resource?key=value"`. It takes an implicit [io.circe.Decoder[T]](https://circe.github.io/circe/api/io/circe/Decoder.html){:target="_blank"} as well, used to decode the json result into `T`.

`get[T]` returns a `zio.Task[T]`, [a type alias for `ZIO[Any, Throwable, T]`](https://zio.dev/docs/overview/overview_index#type-aliases){:target="_blank"}, which represents an effect that has no requirements, and may fail with a `Throwable` value, or succeed with a `T`.

[Following the module recipe](https://zio.dev/docs/howto/howto_use_layers#the-module-recipe){:target="_blank"}, we have:

```scala
type HttpClient = Has[Service]
```

In simple terms, `Has` allows us to use our `Service` as a dependency. The next line makes it easier to understand:

```scala
def http4s: ZLayer[Has[Client[Task]], Nothing, HttpClient] = ???
```

The `http4s` method will create a [`ZLayer`](https://zio.dev/docs/overview/overview_index#zio){:target="_blank"}, which is very similar to [`ZIO` data type](https://zio.dev/docs/overview/overview_index#zio){:target="_blank"}; it requires a `Has[Client[Task]]` to be built, won't produce any errors (that's what that `Nothing` means) and will return an implementation of our Service: `HttpClient`, the one we defined using `Has`.

We should use type aliases to make `ZLayer` more expressive as well. Knowing our layer can't fail, we can use `URLayer`:

```scala
def http4s: URLayer[Has[Client[Task]], HttpClient] = ???
```

*What will `http4s` actually return?* In order to answer this question, we need to implement `HttpClient.Service` first.

## The `Http4s` implementation

Implementing the get request is straightforward:

```scala
package pdt.http

import io.circe.Decoder
import org.http4s.Uri
import org.http4s.circe.CirceEntityCodec.circeEntityDecoder
import org.http4s.client.Client
import org.http4s.client.dsl.Http4sClientDsl
import zio._
import zio.interop.catz._

private[http] final case class Http4s(client: Client[Task])
  extends HttpClient.Service with Http4sClientDsl[Task] {

  def get[T](resource: String, parameters: Map[String, String])
            (implicit d: Decoder[T]): Task[T] = {
    val uri = Uri(path = rootUrl + resource).withQueryParams(parameters)

    client.expect[T](uri.toString())
  }
```

Maybe you are scratching your head due to that `import zio.interop.catz._`. `http4s` is built on top of the Cats Effect stack, therefore we need [the `interop-catz` module for interoperability](https://zio.dev/docs/interop/interop_catseffect){:target="_blank"}.

An instance of this class can't be created outside the `http` package; the instance will be provided through our `ZLayer`. Let's go back to `HttpClient.http4s`, it's time to implement it!

### Providing an `HttpClient.Service` through `ZLayer`

Having a service definition, `ZLayer.fromService` seems appropriate:

```scala
object HttpClient {

  def http4s: URLayer[Has[Client[Task]], HttpClient] =
    ZLayer.fromService[Client[Task], Service] { http4sClient =>
      Http4s(http4sClient)
    }
}
```

Okay, this layer makes our HttpClient available. How can we access it? Let's start defining something that uses the client, a concrete example always makes learning easier :)

### A couple of useful helpers

The first resource, [Acordos de Leniência](http://www.transparencia.gov.br/swagger-ui.html#/Acordos32de32Leni234ncia){:target="_blank"} is a good candidate:

- `GET /acordos-leniencia/{id}` returns an object;
- `GET /acordos-leniencia` (with filters as query params) returns a list of objects;

The rest of the API exposes basically the same for other resources, just with more filters. Knowing that, we can define two helpers, one for each case:

```scala
object HttpClient {
  // ...

  def get[T](resource: String, id: Long)
            (implicit d: Decoder[T]): RIO[HttpClient, T] =
    RIO.accessM[HttpClient](_.get.get[T](s"$resource/$id", Map()))

  def get[T](resource: String, parameters: Map[String, String] = Map())
            (implicit d: Decoder[T]): RIO[HttpClient, List[T]] =
    RIO.accessM[HttpClient](_.get.get[List[T]](resource, parameters))
}
```

> `RIO.accessM[HttpClient]` effectfully accesses the environment of our effect, giving us `Has[HttpClient.Service]`, so we call the first `get` to access the effect wrapped by `Has` - our Service - while the second `get` is the actual get request.

To make it clear, if we had a `post` method, the code would be:

```scala
RIO.accessM[HttpClient](_.get.post[T](resource, parameters))
```

Alright, let's make the whole thing work!

## A concrete HttpClient... client (?!?!?)

Then again, *Acordos de Leniência* is our resource. This is a case class for its possible filters (brazilian api, names in portuguese):

  ```scala
case class AcordoLenienciaRequest(
              cnpjSancionado: Option[String] = None,
              nomeSancionado: Option[String] = None,
              situacao: Option[String] = None,
              dataInicialSancao: Option[LocalDate] = None,
              dataFinalSancao: Option[LocalDate] = None,
              pagina: Int = 1)
  ```

And the response:

```scala
case class AcordoLeniencia(
              id: Long,
              nomeEmpresa: String,
              dataInicioAcordo: LocalDate,
              dataFimAcordo: LocalDate,
              orgaoResponsavel: String,
              cnpj: String,
              razaoSocial: String,
              nomeFantasia: String,
              ufEmpresa: String,
              situacaoAcordo: String,
              quantidade: Int)
```

`AcordosLenienciaClient` couldn't be simpler:

```scala
import io.circe.generic.auto._
import pdt.client.decoders.localDateDecoder
import pdt.http.HttpClient.{HttpClient, get}
import pdt.domain.{AcordoLeniencia, AcordoLenienciaRequest => ALRequest}
import pdt.http.implicits.HttpRequestOps
import zio._

object AcordosLenienciaClient {

  def by(id: Long): RIO[HttpClient, AcordoLeniencia] =
    get[AcordoLeniencia]("acordos-leniencia", id)

  def by(request: ALRequest): RIO[HttpClient, List[AcordoLeniencia]] =
    get[AcordoLeniencia]("acordos-leniencia", request.parameters)
}
```

> The implicit method `HttpRequestOps.parameters` transforms any request into a `Map[String, String]`. [Check it out how I used shapeless to do so](/2020/04/06/shapeless-a-real-world-use-case/){:target="_blank"}.

Now we just need to put all the pieces together, sort out dependencies, this kind of thing. That happens at the end of the world... also known as `Main`.

## ZIO Modules, assemble!

Here is a program that makes a request to get a list of `AcordoLeniencia`:

```scala
val program = for {
  result <- AcordosLeniencia.by(AcordoLenienciaRequest())
  _ <- putStrLn(result.toString())
} yield ()
```

It requires a `ZLayer` that produces an `HttpClient`, which has `Client[Task]` as its own dependency. Let's create the `Client[Task]` as a [managed resource](https://zio.dev/docs/datatypes/datatypes_managed){:target="_blank"} first:

```scala
private def makeHttpClient: UIO[TaskManaged[Client[Task]]] =
  ZIO.runtime[Any].map { implicit rts =>
      BlazeClientBuilder
        .apply[Task](Implicits.global)
        .resource
        .toManaged
    }
```

Now we can sort out the layers:

```scala
val httpClientLayer = makeHttpClient.toLayer.orDie
val http4sClientLayer = httpClientLayer >>> HttpClient.http4s
```

and finally, provide our `program` with the required layer:

```scala
program.provideSomeLayer[ZEnv](http4sClientLayer)
```

Ready to go:

```scala
program.foldM(
  e => putStrLn(s"Execution failed with: ${e.printStackTrace()}") *> ZIO.succeed(1),
  _ => ZIO.succeed(0)
)
```

And this is how I built it. Some code is different here from the original, for learning purposes. [You can find the code on Github](https://github.com/juliano/pdt-client){:target="_blank"}.

Before we jump to the conclusion, let's consolidate what we've learnt adding a new dependency, a `logger` that prints in the console the url requested, and the error message if it fails.

## Adding a new dependency, step by step

The `Logger` module definition:

```scala
object Logger {
  type Logger = Has[Service]

  trait Service {
    def info(message: => String): UIO[Unit]
    def error(t: Throwable)(message: => String): UIO[Unit]
  }
}
```

The implementation, printing to the console:

```scala
import zio.console.{Console => ConsoleZIO}

case class Console(console: ConsoleZIO.Service)
  extends Logger.Service {

  def info(message: => String): UIO[Unit] =
    console.putStrLn(message)

  def error(t: Throwable)(message: => String): UIO[Unit] =
    for {
      _ <- console.putStrLn(message)
      _ <- console.putStrLn(t.stackTrace)
    } yield ()
}
```

`Logger` makes the implementation available via `ZLayer`:

```scala
object Logger {

  def console: URLayer[ConsoleZIO, Logger] =
    ZLayer.fromService[ConsoleZIO.Service, Service] { console =>
      Console(console)
    }
}
```

`Http4s` can now receive and use a `logger` instance:

```scala
private[http] final case class Http4s(logger: Logger.Service, client: Client[Task])
  extends HttpClient.Service with Http4sClientDsl[Task] {

  def get[T](resource: String, parameters: Map[String, String])
            (implicit d: Decoder[T]): Task[T] = {
    val uri = Uri(path = rootUrl + resource).withQueryParams(parameters)

    logger.info(s"GET REQUEST: $uri") *>
      client
        .expect[T](uri.toString())
        .foldM(
          e => logger.error(e)("Request failed") *> IO.fail(e),
          ZIO.succeed(_))
  }
}
```

The `http4s` layer needs to adapt:

```scala
object HttpClient {

  def http4s: URLayer[Logger with Has[Client[Task]], HttpClient] =
    ZLayer.fromServices[Logger.Service, Client[Task], Service] {
      (logger, http4sClient) =>
        Http4s(logger, http4sClient)
    }
}
```

Let's feed our `program` with the new dependency. The change is in the layer provided:

```scala
val http4sClientLayer = (loggerLayer ++ httpClientLayer) >>> HttpClient.http4s
```

Done!

## Summary

My first experience with ZIO has been very pleasant. In order to solve dependencies, the compiler plays on our side; every time something is missing, we have an error in compile time, with a clear indication of what is missing. Besides, ZLayer makes dependency resolution extremely simple and extensible (think about adding a `FileLogger` for example) without magic.

Any suggestions to improve that code? Please share!

### References

- [ZIO - Using modules and Layers](https://zio.dev/docs/howto/howto_use_layers)
- [Solving the Dependency Injection Problem with ZIO](https://github.com/adamgfraser/solving-the-dependency-injection-problem-with-zio/blob/master/solving-the-dependency-injection-problem-with-zio.pdf)
- [From idea to product with ZLayer](https://scala.monster/welcome-zio/)
- [ZIO, Http4s, Auth, Codecs and zio-test](https://timpigden.github.io/_pages/zio-http4s/intro.html)
- [Http4s Client](https://http4s.org/v0.21/client/)
