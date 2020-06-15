---
layout: single
title: "Streaming all the way with ZIO, Doobie, Quill, http4s and fs2"
header:
  teaser: /assets/images/cascade.jpg
  imgcredit: Image by Pixabay from Pexels
categories:
  - scala
  - zio
  - doobie
  - quill
  - http4s
  - fs2
---

Data is flowing nonstop in the real world, as much as in the digital one. In this post we will see how one can make static data from the database surf the streaming wave and :warning: *BAD JOKE ALERT* :warning: go with the flow.

Probably "surf the streaming wave" needed an warning as well.

## The Configurations

Pretty standard `h2` database and the http server configurations:

```conf
database {
  driver = "org.h2.Driver"
  url = "jdbc:h2:./example;DB_CLOSE_DELAY=-1"
  user = ""
  password = ""
}
http-server {
  host = "0.0.0.0"
  path = "/api/v1"
  port = 8080
}
```

And the respective `object Configuration`, using `pureconfig`:

```scala
import pureconfig.generic.auto._

object Configuration {
  final case class AppConfig(database: DbConfig, httpServer: HttpServerConfig)
  final case class DbConfig(driver: String, url: String, user: String, password: String)
  final case class HttpServerConfig(host: String, port: Int, path: String)
}
```

## The Datasource

Our database has all the cities in the world.

```scala
case class City(
  id: Int,
  name: String,
  countryCode: String,
  district: String,
  population: Int
)
```

`CitiesRepository.Service` will stream them all, or all the cities from a given country.

```scala
object CitiesRepository {
  trait Service {
    def all: fs2.Stream[Task, City]
    def byCountry(country: String): fs2.Stream[Task, City]
  }
}
```

Having that well defined, here is where the fun begins.

## Composable database queries by [Quill](https://getquill.io/){:target="_blank"}

Instead of writing the SQL queries ourselves, let's write some vanilla Scala and Quill will do the heavy lifting:

```scala
val cities = quote(query[City])
// SELECT x.id, x.name, x.countryCode, x.district, x.population FROM City x

def citiesByCountry(country: String) = quote {
  cities.filter(_.countryCode == lift(country))
}
// SELECT x.id, x.name, x.countryCode, x.district, x.population
//   FROM City x WHERE x.countryCode = $1
```

If you are new to Quill, you can find [a lot of content in the blog](/categories/#quill). I know, [I talk a lot about Quill](/speaking)! :wink:

## Composable database interactions by [Doobie](https://tpolecat.github.io/doobie/){:target="_blank"}

Doobie provides [the stream capability our application demands](https://tpolecat.github.io/doobie/docs/04-Selecting.html#internal-streaming){:target="_blank"}, emitting rows as they arrive from the database via `fs2.Stream`. Moreover, [it easilty integrates with Quill](https://tpolecat.github.io/doobie/docs/17-Quill.html){:target="_blank"}. What a beautiful match!

```scala
private final case class Database(xa: Transactor[Task])
    extends CitiesRepository.Service {
  val ctx = new DoobieContext.H2(Literal)
  import ctx._

  def all: fs2.Stream[Task, City] =
    ctx.stream(cities).transact(xa)

  def byCountry(country: String): fs2.Stream[Task, City] =
    ctx.stream(citiesByCountry(country)).transact(xa)

  val cities = quote(query[City])

  def citiesByCountry(country: String) = quote {
    cities.filter(_.countryCode == lift(country))
  }
}
```

> Is this integration new to you? Learn more about it from [Quill's lead mainteiner Alexander loffe](https://twitter.com/deusaquilus){:target="_blank"} himself watching [Quill + Doobie = Better Together](https://youtu.be/1WVjkP_G2cA){:target="_blank"}

Doobie needs a `Transactor` to do its job, and `Transactor` needs a `DbConfig` to be created. In order to handle that, we will define a `ZLayer` in the `package object repository`:

```scala
package object repository {
  type DbTransactor = Has[DbTransactor.Resource]

  object DbTransactor {
    trait Resource {
      val xa: Transactor[Task]
    }

    val h2: URLayer[Has[DbConfig], DbTransactor] =
      ZLayer.fromService { db =>
        new Resource {
          val xa: Transactor[Task] =
            Transactor.fromDriverManager(
              db.driver, db.url, db.user, db.password
            )
        }
      }
  }
}
```

We are already here, so why not adding a few more utilities?

```scala
type CitiesRepository = Has[CitiesRepository.Service]

def allCities: RIO[CitiesRepository, fs2.Stream[Task, City]] =
  RIO.access(_.get.all)

def citiesByCountry(country: String): RIO[CitiesRepository, fs2.Stream[Task, City]] =
  RIO.access(_.get.byCountry(country))
```

> Aren't you familiar with ZLayer yet? Fear nothing! [Managing dependencies using ZIO](https://blog.softwaremill.com/managing-dependencies-using-zio-8acc1539e276){:target="_blank"} by [Adam Warski](https://twitter.com/adamwarski){:target="_blank"}

Can you feel it, dear reader? It's the power of data flowing. Now let's direct it to the outside world!

## The Endpoint

You've probably seen [Wiem Zine post](https://medium.com/@wiemzin/zio-with-http4s-and-doobie-952fba51d089){:target="_blank"} (read it if you haven't yet!). We will describe our routes in a similar way.

I find this section the most complex, there are some gotchas here. Have a look at `CitiesEndpoint`:

```scala
final class CitiesEndpoint[R <: CitiesRepository] {
  type CitiesTask[A] = RIO[R, A]

  private val prefixPath = "/cities"

  val dsl = Http4sDsl[CitiesTask]
  import dsl._

  implicit def cityEncoder[A](implicit encoder: Encoder[A]):
    EntityEncoder[CitiesTask, A] = jsonEncoderOf[CitiesTask, A]

  val routes: HttpRoutes[CitiesTask] = Router(
    prefixPath -> httpRoutes
  )

  private val httpRoutes = HttpRoutes.of[CitiesTask] {
    ???
  }
}
```

Let's apply some simplifications first. When looking at `CitiesTask[A]`, the `A` commonly would be a `City`, a `Option[City]` or `List[City]`, but in our case, after expanded it will be a

```scala
fs2.Stream[RIO[CitiesRepository, City], City]`
```

It could be nicer. The equivalent type `CitiesTask` makes it more readable:

```scala
fs2.Stream[CitiesTask, City]
```

I would say this concept is important enough to deserve it's own type:

```scala
type CitiesStream = fs2.Stream[CitiesTask, City]
```

Let's leve it here for a second, while we define our route:

```scala
import io.circe.syntax._

private val httpRoutes = HttpRoutes.of[CitiesTask] {
  case GET -> Root =>
    for {
      stream <- allCities
      json <- Ok(stream.map(_.asJson))
    } yield json
}
```

Unfortunately, it fails with:

```shell
Error:(26, 52) Cannot convert from fs2.Stream[[+A]zio.ZIO[Any,Throwable,A],io.circe.Json] to an Entity, because no EntityEncoder[[A]zio.ZIO[R,Throwable,A], fs2.Stream[[+A]zio.ZIO[Any,Throwable,A],io.circe.Json]] instance could be found.
      allCities.flatMap(stream => Ok(stream.map(_.asJson)))
```

Remember I mentioned there are a couple of gotchas here?

### Gotcha #1: Help the compiler to help you

Despite the error message, everything we need is already in place, but the compiler is a bit... confused. It needs a hint, so we will make a small change in the code:

```scala
case GET -> Root =>
  val pipeline: CitiesTask[CitiesStream] = allCities
  for {
    stream <- allCities
    json <- Ok(stream.map(_.asJson))
  } yield json
```

It compiles now! Let's define the other route, which takes a `country` query parameter. This time I used `flatMap`, in order to illustrate that we still have to give the compiler a hint. The whole `httpRoutes` is:

```scala
object CountryParameter extends
    QueryParamDecoderMatcher[String]("country")

private val httpRoutes = HttpRoutes.of[CitiesTask] {
  case GET -> Root :? CountryParameter(country) =>
    val pipeline: CitiesTask[CitiesStream] = citiesByCountry(country)
    pipeline.flatMap(stream => Ok(stream.map(_.asJson)))

  case GET -> Root =>
    val pipeline: CitiesTask[CitiesStream] = allCities
    for {
      stream <- pipeline
      json <- Ok(stream.map(_.asJson))
    } yield json
}
```

What else can go wrong?

### Gotcha #2: Have the right imports in place

We need some stream enconders in place, which are provided by `http4s-circe`, but it's easy to miss them because the compiler won't complain if they are not there:

```scala
import org.http4s.circe._
```

Without this `import`, the serialization won't work as expected and it will break on the client side.

## The Server

The server implementation is fairly straightforward (assuming you are familiar with [zio + http4s](https://timpigden.github.io/_pages/zio-http4s/intro.html){:target="_blank"}):

```scala
object Server {
  type ServerRIO[A] = RIO[AppEnvironment, A]
  type ServerRoutes =
    Kleisli[ServerRIO, Request[ServerRIO], Response[ServerRIO]]

  def runServer: ZIO[AppEnvironment, Nothing, Unit] =
    ZIO.runtime[AppEnvironment].flatMap { implicit rts =>
      val cfg = rts.environment.get[HttpServerConfig]
      val ec = rts.platform.executor.asEC

      BlazeServerBuilder[ServerRIO](ec)
        .bindHttp(cfg.port, cfg.host)
        .withHttpApp(createRoutes(cfg.path))
        .serve
        .compile[ServerRIO, ServerRIO, ExitCode]
        .drain
    }
      .orDie

  def createRoutes(basePath: String): ServerRoutes = {
    val citiesRoutes = new CitiesEndpoint[AppEnvironment].routes
    val healthRoutes = new HealthEndpoint[AppEnvironment].routes
    val routes = citiesRoutes <+> healthRoutes

    Router[ServerRIO](basePath -> routes).orNotFound
  }
}
```

"Hey! `AppEnvironment` is not declared anywhere!" - I know, you don't need to shout! That will be happen in the next section!

## Putting everything together

First, we need a `ZLayer` with the `Configuration`:

```scala
type Configuration = Has[DbConfig] with Has[HttpServerConfig]

object Configuration {
  val live: ULayer[Configuration] = ZLayer.fromEffectMany(
    ZIO
      .effect(ConfigSource.default.loadOrThrow[AppConfig])
      .map(c => Has(c.database) ++ Has(c.httpServer))
      .orDie
  )
}
```

Everything the app needs will be placed in `Environments`:

```scala
object Environments {
  type HttpServerEnvironment = Configuration with Clock
  type AppEnvironment = HttpServerEnvironment with CitiesRepository

  val httpServerEnvironment: ULayer[HttpServerEnvironment] =
    Configuration.live ++ Clock.live

  val dbTransactor: ULayer[DbTransactor] =
    Configuration.live >>> DbTransactor.h2

  val citiesRepository: ULayer[CitiesRepository] =
    dbTransactor >>> CitiesRepository.live

  val appEnvironment: ULayer[AppEnvironment] =
    httpServerEnvironment ++ citiesRepository
}
```

We only have a server to run, so our `Main` has only a few lines:

```scala
object Main extends App {
  def run(args: List[String]): ZIO[ZEnv, Nothing, ExitCode] = {
    val program = for {
      _ <- Server.runServer
    } yield ()

    program.provideLayer(appEnvironment).exitCode
  }
}
```

No one can stop it now. The data is flowing!

![](/assets/images/kraken.jpg){: .align-center}

## Summary

You are now one step closer to write your own Netflix :tv:!

I wrote this post intending to have an example at the end more than a tutorial, reason why I don't spend any time explaining how the tools work, assuming the reader already has relative good knowledge of them. Even though this is a very simplified streaming engine, the same principles and tools could be used to build something much bigger!

[You can find the code for this example here.](https://github.com/juliano/streaming-all-the-way)
