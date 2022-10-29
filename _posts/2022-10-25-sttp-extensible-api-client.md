---
layout: single
title: "Sttp: An Extensible API client"
header:
  teaser: /assets/images/poke-space-station.jpg
  imgcredit: Image by PIRO from Pixabay
categories:
  - sttp
  - zio-json
  - opensource
classes: wide
---

A while back I blogged about [how to write an api client using ZIO and http4s client](/2020/04/20/zio-http4s-a-simple-api-client). Revisiting it is interesting, so much changed since then! The post was a nice example at the time, but thinking about that solution as a real world implementation, it suffers from a couple of problems:

* it's not extensible;
* it's highly coupled - what is not necessarily a problem, when it's part of the domain. But it would be nice to have.

Simply making changes according to these two topics wouldn't evolve the idea that much though, so here's the new proposition: let's implement an extensible client, properly!

## What's wrong with the previous client?

Having `client.getThis` and `client.getThat` implicates in changing the client itself whenever we need to add a new request. Besides, the [request / response strategy](/2020/04/20/zio-http4s-a-simple-api-client/#a-concrete-httpclient-client-) could be better and shapeless should be removed.

Let's start with a base definition for our new requests:

```scala3
Request[Response]
```

This is a brilliant approach that I "borrowed" from [sttp](https://sttp.softwaremill.com/en/latest/){:target="_blank"}, where the request carries the information about the response. In principle, we could define a trait with one single method:

```scala
def send[A](request: Request[A]): F[A]
```

For the new, modern, super-duper awesome client, let's keep the effect system generic. Just for fun :wink:

## What is the plan?

We need a public API to consume, so we will build a client to the [pokeapi.co](https://pokeapi.co/){:target="_blank"}. If you haven't watched Pokemon, don't tell me. That would make me feel old!

<a href="https://pokeapi.co/">
![](/assets/images/pokemon.png){: .align-center}
</a>

Let's use the already mentioned sttp to send requests, and for JSON parsing we'll be using [zio-json](https://zio.github.io/zio-json/){:target="_blank"}.

## Let's catch them all!

First things first, an api host:

```scala
import sttp.model.Uri
import sttp.model.Uri.UriContext

final case class ApiHost private (uri: Uri)

object ApiHost:
  final val default = ApiHost(uri"https://pokeapi.co/api/v2")
```

Now let's define the generic request to consume the Pokemon Api. I know it's not so obvious, but what we need is the...

### PokeRequest (Best. Name. Ever.)

Most resources available via pokeapi.co can be requested using the following pattern:

https://pokeapi.co/api/v2/**{resource}**/**{id or name}**/

Let's abstract the path parameters and make it easy for the user to create a request in the following format, so it can be sent via sttp:

```scala
Request[Either[ResponseException[String, String], A], Any]
```

A request that can fail with a `ResponseException` or succeed with `A`... 🤔 this is not that readable, is it? I think a couple of type aliases will be handy here:

```scala
type FailureResponse = ResponseException[String, String]
type SttpRequest[A]  = Request[Either[FailureResponse, A], Any]
```

That's better, with that we can have a `makeRequest` that returns a `SttpRequest[A]`. It will receive the complete `uri` and a `JsonDecoder[A]` implicitly:

```scala
import sttp.client3.ziojson.asJson
import sttp.model.{ MediaType, Uri }
import zio.json.JsonDecoder

def makeRequest[A](uri: Uri)(using JsonDecoder[A]): SttpRequest[A] =
  basicRequest
    .get(uri)
    .readTimeout(10.seconds)
    .contentType(MediaType.ApplicationJson)
    .response(asJson[A])
```

The complete way to build requests using sttp is [very well documented here](https://sttp.softwaremill.com/en/latest/requests/basics.html){:target="_blank"}. `makeRequest` returns the description of a simple `GET` request that parses the result into json using `JsonDecoder`.

Now we have all we need to create the trait `PokeRequest`:

```scala
trait PokeRequest[A](id: String | Long):
  val resource: String

  def sttpRequest(host: ApiHost)(using JsonDecoder[A]): SttpRequest[A] =
    makeRequest(host.uri.addPath(resource, id.toString))
```

The new trait is decoupled from the client, and easily customizable, well done us! Before creating a concrete request we need the response well defined (the generic **A** in the request), so let's define a `Berry`:

```scala
import zio.json.{ jsonField, DeriveJsonDecoder, JsonDecoder }

final case class Berry(
    id: Int,
    name: String,
    @jsonField("growth_time") growthTime: Int,
    @jsonField("max_harvest") maxHarvest: Int,
    @jsonField("natural_gift_power") naturalGiftPower: Int,
    size: Int,
    smoothness: Int,
    @jsonField("soil_dryness") soilDryness: Int,
    firmness: NamedAPIResource,
    flavors: List[BerryFlavorMap],
    item: NamedAPIResource,
    @jsonField("natural_gift_type") naturalGiftType: NamedAPIResource
)

object Berry:
  given JsonDecoder[Berry] = DeriveJsonDecoder.gen
```

The pokeapi response has all its field names in snake case, so `jsonField` is used when the names don't match. The most important part is the [creation of the `JsonDecoder` using the code generator](https://github.com/zio/zio-json#simple-example){:target="_blank"}. All in place to write `BerryRequest`:

```scala
final case class BerryRequest(id: String | Long) extends PokeRequest[Berry](id):
  val resource = "berry"
```

Having the response, writting the request is that simple! What do we need now to send request? You won't see that coming, dear reader! We need a...

### PokeApiClient

> I am a master naming stuff

Cool, now let's get to business. Sttp gives us the generic structures we are looking for, we just need to adapt it to our domain, making it more restrict. Our `PokeApiClient` will be defined using an `SttpBackend`:

```scala
import sttp.client3.*
import sttp.monad.MonadError

case class PokeApiClient[F[_], +P](host: ApiHost)(using
    backend: SttpBackend[F, P]
):
  given monadError: MonadError[F] = backend.responseMonad
```

The `backend` wraps the real http client, `F` is the effect type in use (`ZIO`, `Future`, etc) and `P` indicates extra capabilities like streaming or websockets, so don't worry about it, we are not using any of those. The last important bit is the `given monadError`: it's the way to inform if the request was successful or not.

The client will have only one public method `send` that receives the request, the `JsonDecoder` and return a `F[A]`. However, splitting it in two makes it easier to understand:

```scala
def send[A](request: PokeRequest[A])(using JsonDecoder[A]): F[A] =
  doSend(request).flatMap {
    case Right(value) => monadError.unit(value)
    case Left(error) =>  monadError.error(error)
  }

private def doSend[A](
    request: PokeRequest[A]
)(using JsonDecoder[A]): F[Either[FailureResponse, A]] =
  request
    .sttpRequest(host)
    .send(backend)
    .map(_.body)
```

`doSend` provides the request with a host and effectively sends the request using the backend, returning the response body. `send` will then `flatMap` it and use the appropriate channels of `monadError` to return the result, `unit` for success and `error` for failure.

This last line some complexity in it, but honestly, don't think about it. Let's focus on what's matter: we have a generic, extensible api client!

### Using the PokeApiClient

All the [sttp backends](https://sttp.softwaremill.com/en/latest/backends/summary.html){:target="_blank"} published for Scala 3 are supported. An example with `Future`s:

```scala
given backend: SttpBackend[Future, Any] = HttpClientFutureBackend()
val client = PokeApiClient()

client.send(ContestTypeRequest(1)).onComplete {
    case Success(contest) => println(contest.names)
    case Failure(t)       => println(s"Failed with: $t")
}
```

or with ZIO:

```scala
val client = AsyncHttpClientZioBackend().map(implicit backend => PokeApiClient())

val zio = client.flatMap(_.send(PokemonRequest("bulbasaur")))
val pokemon = Unsafe.unsafeCompat { implicit u =>
  Runtime.default.unsafe.run(zio).getOrThrowFiberFailure()
}
print(pokemon.id)
```

You can find more examples using different backends, and [the complete code in the Github repo](https://github.com/juliano/pokeapi-scala){:target="_blank"}.

## Summary

Revisiting our own code is a stimulating experience - always good to see how the perspective changed, and what we've learned. Sttp is a powerful tool, and the knowledge acquired studying it made it easy to create an extensible api client, that allows the user to decide which effect system they want to use.

The final outcome is in the list of [wrapper libraries of the PokeApi](https://pokeapi.co/docs/v2#wrap){:target="_blank"}. Can you see possible improvements? Just let me know!

### References

- [sttp: the Scala HTTP client you always wanted!](https://sttp.softwaremill.com/en/latest/){:target="_blank"}
- [Fast, secure JSON library with tight ZIO integration.](https://github.com/zio/zio-json){:target="_blank"}
