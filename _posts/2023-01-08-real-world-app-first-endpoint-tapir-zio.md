---
layout: single
title: "Real World App: First endpoint - Tapir and ZIO"
header:
  teaser: /assets/images/doors.jpg
  imgcredit: Image by Arek Socha from Pixabay
categories:
  - tapir
  - zio
  - opensource
  - real-world-app
classes: wide
---

It is happenning. This is the first post of a series where we will build together THE BEST version of [the Real Word App](https://realworld-docs.netlify.app/docs/specs/backend-specs/introduction/){:target="_blank"} there is, thanks to the wonderful scala stack we have available. There's a lot of content to cover dear reader, it will be a long journey. Are you ready? I said, are you readyyyy???

![](/assets/images/rumble.jpeg){: .align-center}

## Preparing the ground

The goal of this post is to have a functional http server, exposing an endpoint that can handle json. Easy? Let's see. Here are the dependencies we need to get started:

```scala
libraryDependencies ++= Seq(
  "dev.zio"                     %% "zio"                   % "2.0.3",
  "com.softwaremill.sttp.tapir" %% "tapir-zio"             % "1.2.4",
  "com.softwaremill.sttp.tapir" %% "tapir-json-zio"        % "1.2.4",
  "com.softwaremill.sttp.tapir" %% "tapir-zio-http-server" % "1.2.4"
)
```

I am thinking of taking it easy at the beginning. [Get Current User endpoint](https://realworld-docs.netlify.app/docs/specs/backend-specs/endpoints/#get-current-user) looks like a good starting point, it's a simple `GET` request that returns an [User](https://realworld-docs.netlify.app/docs/specs/backend-specs/api-response-format/#users-for-authentication). According to the definition, let's create our model `User`:

```scala
package io.github.juliano.realworld.models

import sttp.tapir.Schema
import zio.json.{ DeriveJsonCodec, JsonCodec }

final case class User(
    email: String,
    token: String,
    username: String,
    bio: Option[String],
    image: Option[String]
)

object User:
  given JsonCodec[User] = DeriveJsonCodec.gen
  given Schema[User] = Schema.derived
```

The `given` instace of `JsonCodec[User]` has the ability to encode/decode an `User` to/from json. Don't worry about creating that instance yourself, `DeriveJsonCodec.gen` will make the magic happen:

```bash
scala> import zio.json.*

scala> val user = User("juliano@email.com", "t0ken", "juliano", None, None)
val user: User = User(juliano@email.com,t0ken,juliano,None,None)

scala> val json = user.toJson
val json: String = {"email":"juliano@email.com","token":"t0ken","username":"juliano"}

scala> json.fromJson[User]
val res0: Either[String, User] = Right(User(juliano@email.com,t0ken,juliano,None,None))
```

Almost there, the generated json should be wrapped by an object `"user" : {...}`, according to the documentation. We can define an `UserResponse` for that:

```scala
package io.github.juliano.realworld.models.api

import io.github.juliano.realworld.models.User
import sttp.tapir.Schema
import zio.json.{ DeriveJsonCodec, JsonCodec }

final case class UserResponse(user: User)

object UserResponse:
  given JsonCodec[UserResponse] = DeriveJsonCodec.gen
  given Schema[UserResponse] = Schema.derived
```

Using the console again to see it by ourselves, we have:

```bash
scala> UserResponse(user).toJson
val res2: String = {"user":{"email":"juliano@email.com","token":"t0ken","username":"juliano"}}
```

Alongside the `JsonCodec` we've got an instance of `Schema[User]`. The schema describes the mapping between low-level (`String`) and high-level (`User`) values, used to encoded, decode and validate. We need it in place for the next step, so just trust me! ... or you can [read more about codecs and schemas here](https://tapir.softwaremill.com/en/latest/endpoint/codecs.html){:target="_blank"}, and about [schema derivation here](https://tapir.softwaremill.com/en/latest/endpoint/schemas.html){:target="_blank"}, it's your choice! :nerd_face:

Now let's create our first endpoint!

## The first endpoint: `GET /user`

Tapir's declarative way to define endpoints makes it very enjoyable to work with. We need an endpoint that accepts a `GET /api/user` and responds with an `UserResponse` in json format. The code implementing this description translates pretty much literally:

```scala
package io.github.juliano.realworld.server.endpoints

import io.github.juliano.realworld.models.api.UserResponse
import sttp.tapir.Endpoint
import sttp.tapir.json.zio.jsonBody
import sttp.tapir.ztapir.*
import zio.*

final case class UserEndpoint():
  val getEndpoint: Endpoint[Unit, Unit, Unit, UserResponse, Any] =
    endpoint.get
      .in("api" / "user")
      .out(jsonBody[UserResponse])

  val get: ZServerEndpoint[Any, Any] =
    getEndpoint.zServerLogic(_ =>
      ZIO.succeed {
        UserResponse(User("juliano@email.com", "t0ken", "juliano", None, None))
      }
    )

  val routes: List[ZServerEndpoint[Any, Any]] = List(get)
```

`getEndpoint` describes in code what has been mentioned above. Here is the `Endpoint` definition, naming every type parameter it takes:

```scala3
Endpoint[SECURITY_INPUT, INPUT, ERROR_OUTPUT, OUTPUT, -CAPABILITIES]
```

I could explain endpoints declaration in details, but the Tapir creators, having your best interests in mind, [have already done so in the docs](https://tapir.softwaremill.com/en/latest/endpoint/basics.html#basics){:target="_blank"} :wink:.

Well, the declaration by itself doesn't do much, that's why we need to [provide it with a function that implements the server logic](https://tapir.softwaremill.com/en/latest/server/logic.html#server-logic){:target="_blank"} using `zServerLogic`. The function here matches what is defined in the `getEndpoint`: there is no input (represented by `Unit` that we ignore), and returns a `ZIO[Any, Unit, UserResponse]`.

The combination of the endpoint description with the server function gives us a `ZServerEndpoint[Any, Any]` where the first parameter `Any` refers to the `ZIO` environment (the `R`), and the second one to `CAPABILITIES` mentioned above, the last parameter of `Endpoint`. All the `ZServerEndpoints` defined will be added to `routes`, making it available for the server. Speaking of availability, there's one last thing we need:

```scala
object UserEndpoint:
  val layer: ULayer[UserEndpoint] = ZLayer.succeed(UserEndpoint())
```

The time has come. Let's create the http server and expose what we've built!

## Exposing endpoints with [zio-http](https://zio.dev/zio-http/){:target="_blank"} server

The `RealWorldServer` is fairly straightforward, it will put all the routes together (just one for now), use Tapir [ZioHttpInterpreter](https://tapir.softwaremill.com/en/latest/server/ziohttp.html){:target="_blank"} to transform a list of `ZServerEndpoint[Any, Any]` into a `HttpApp[Any, Throwable]`, that can be served by the zio `Server`:

```scala
package io.github.juliano.realworld.server

import io.github.juliano.realworld.server.endpoints.*
import sttp.tapir.server.ziohttp.ZioHttpInterpreter
import zio.*
import zio.http.{ HttpApp, Server }

final case class RealWorldServer(userEndpoint: UserEndpoint):
  val allRoutes = userEndpoint.routes

  val serverRoutes: HttpApp[Any, Throwable] =
    ZioHttpInterpreter().toHttp(allRoutes)

  def start: Task[Nothing] = Server.serve(serverRoutes).provide(Server.default)

object RealWorldServer:
  val layer: URLayer[UserEndpoint, RealWorldServer] =
    ZLayer.fromFunction(RealWorldServer.apply _)
```

All pieces are there, let's just put them to work together:

```scala
package io.github.juliano.realworld

import io.github.juliano.realworld.server.RealWorldServer
import io.github.juliano.realworld.server.endpoints.*
import zio.{ ZIO, ZIOAppDefault }

object RealWorldApp extends ZIOAppDefault:
  def run =
    ZIO
      .serviceWithZIO[RealWorldServer](_.start)
      .provide(
        RealWorldServer.layer,
        UserEndpoint.layer
      )
```

The moment is now, dear reader. Will it work? Run the app via sbt and access http://localhost:8080/api/user via `curl` or web browser. The result is:

```json
{
  "user": {
    "email": "juliano@email.com",
    "token": "t0ken",
    "username": "juliano"
  }
}
```

![](/assets/images/victory-is-mine.jpeg){: .align-center}

I mean, ours! :sweat_smile:

## Bonus endpoint: `POST /user`

Now that all the heavy lifting is done, adding a new endpoint will be easy-peasy. The [registration endpoint](https://realworld-docs.netlify.app/docs/specs/backend-specs/endpoints/#registration){:target="_blank"} receives an `user` with fields `username`, `email` and `password`:

```scala
package io.github.juliano.realworld.models.api

import sttp.tapir.Schema
import zio.json.{ DeriveJsonCodec, JsonCodec }

final case class UserCreate(user: UserCreate.Input)

object UserCreate:
  given JsonCodec[UserCreate] = DeriveJsonCodec.gen
  given Schema[UserCreate] = Schema.derived

  final case class Input(email: String, password: String, username: String)
  object Input:
    given JsonCodec[Input] = DeriveJsonCodec.gen
    given Schema[Input] = Schema.derived

```

It is a `POST` that receives an `UserCreate` body and returns an `UserResponse`. Let's add to `UserEndpoint` the respective code:

```scala
// previous imports
import io.github.juliano.realworld.models.api.UserCreate

final case class UserEndpoint():
  // previous vals

  val createEndpoint: Endpoint[Unit, UserCreate, Unit, UserResponse, Any] =
    endpoint.post
      .in("api" / "users")
      .in(jsonBody[UserCreate])
      .out(jsonBody[UserResponse])

  val create: ZServerEndpoint[Any, Any] =
    createEndpoint.zServerLogic(req =>
      ZIO.succeed {
        UserResponse(User("newuser@email.com", "new!", "newuser", None, None))
      }
    )

  val routes: List[ZServerEndpoint[Any, Any]] = List(get, create)
```

Running the curl request:
```bash
curl --request POST 'http://localhost:8080/api/users' \
     --data-raw '{
         "user": {
             "email": "batman@email.com",
             "password": "nananana",
             "username": "batman"
         }
     }' | jq
```
we will see the expected result:
```json
{
    "user": {
        "email": "newuser@email.com",
        "token": "new!",
        "username": "newuser"
    }
}
```
As promised: easy-peasy!

## Next steps
