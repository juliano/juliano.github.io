---
layout: single
title: Building the Death Star with ZIO Stream
header:
  teaser: /assets/images/deathstar.jpg
  imgcredit: Image by Alex_K_83 from Pixabay
categories:
  - scala
  - zio
  - zio-stream
  - streaming
---

Alright team, the current government - you know, [the Galatic Empire](https://starwars.fandom.com/wiki/Galactic_Empire){:target="_blank"} - has made us responsible for their new project, a space station called "Death Star" (what a tacky name). Basically, they want a station literally the size of a moon. If you ask me, [Governor Tarkin](https://starwars.fandom.com/wiki/Wilhuff_Tarkin){:target="_blank"} is trying to show some service.

Anyways. They know we are the best in the galaxy when it comes to organize working streams through our futuristic [`ZIO Stream technology`](https://zio.dev/docs/datatypes/datatypes_stream){:target="_blank"} - and they are right. Let's have a look at the requirements.

## Unfinished Space Station

Have I mentioned that, besides been the size of a moon, it has to be shaped like one? That's the first requirement. Other than that, everything else is pretty much standard, with an exception. The space station requires:

- a Moon Shaped body
- a Propulsion Engine
- Force Shields
- fleet of Tie Fighters
- a **Super Laser** (?!?!?!?)

At the very end, the Empire will have someone here to inspect and approve the project. Here is our end goal:

```scala
case class DeathStar(engine: PropulsionEngine,
                     shield: ForceShield,
                     fleet: List[TieFighter],
                     laser: SuperLaser)
```

Tarkin wants to keep the project in secret (sure - imagine what the taxpayer would say if they knew how their taxes are being used), so we should use a diferent name while it's being built. Let's call it `UnfinishedSpaceStation`:

```scala
case class UnfinishedSpaceStation(
  engine: Option[PropulsionEngine] = None,
  shield: Option[ForceShield] = None,
  fleet: List[TieFighter] = List.empty,
  laser: Option[SuperLaser] = None) {

  def installEngine(engine: PropulsionEngine): UnfinishedSpaceStation =
    copy(engine = Some(engine))

  def installShield(shield: ForceShield): UnfinishedSpaceStation =
    copy(shield = Some(shield))

  def deployFleet(fleet: List[TieFighter]): UnfinishedSpaceStation =
    copy(fleet = fleet)

  def installLaser(superLaser: SuperLaser): UnfinishedSpaceStation =
    copy(laser = Some(superLaser))
}
```

Seems appropriate. Defining how to install/deploy every component is the easy part, the challenge will be to managed and align all the providers of every component.

The Empire is aware of the unknows, expecting it can take some time and possibly, unexpected random problems. So we should make it part of our contract:

```scala
type DStream = ZStream[Clock, Throwable, UnfinishedSpaceStation]
```

Every one of our streams will be a `ZStream` that depends on time via `Clock`, can fail, represented by `Throwable`, and produce an `UnfinishedSpaceStation`. `DStream` is just a shortcut, which makes it easier to comunicate our needs among our providers, so we will use it initially.

Time to turn on the [holoprojector](https://starwars.fandom.com/wiki/Holoprojector){:target="_blank"} and start some conversations. First things first, we don't have an unfinished station without a body.

## The initial source - `SpaceStationBodyShop`

Am I the only one surprised that they already had a production line of "moon shaped stations"? Maybe these days they are more popular than I imagined. Obviously it can take some time to build such a big structure, but at least we don't need any customization from this shop.

```scala
class SpaceStationBodyShop(buildTime: Duration) {

  val moonShapedSpaceStations: DStream =
    ZStream.repeatEffectWith(
      ZIO(UnfinishedSpaceStation()),
      Schedule.spaced(buildTime)
    )
}
```

Towing a moon around is not the easiest task, so adding the `PropulsionEngine` is the next logical step.

## Aligning streams - `PropulsionEngineShop`

Looking at the engines, the options are:

```scala
sealed trait PropulsionEngine

case object HyperDriveEngine extends PropulsionEngine
case object InfiniteImprobabilityDrive extends PropulsionEngine
```

We will order a `HyperDriveEngine`. Never heard of this `InfiniteImprobabilityDrive`, it seems like [something from another universe](https://hitchhikers.fandom.com/wiki/Infinite_Improbability_Drive){:target="_blank"}. This shop is very efficient, we can order a `PropulsionEngine` and they start producing it:

```scala
val engines: Stream[Nothing, PropulsionEngine] = Stream(engine).forever
```

This time we need to define a workflow. `PropulsionEngineShop` will receive a `DStream`, so they need to align their stream of engines with our stream of unfinished stations, install the engine, and return another `DStream`. It can be done using `zip` and `map`:

> `ZStream#zip` - Zips this stream together with the specified stream

> `ZStream#map` - Returns a stream made of the elements of this stream transformed with `f0` (a given function)

```scala
class PropulsionEngineShop(engine: PropulsionEngine) {
  val engines: Stream[Nothing, PropulsionEngine] = Stream(engine).forever

  def installEngine(stations: DStream): DStream =
    stations
      .zip(engines)
      .map { case (station, engine) => station.installEngine(engine) }
}
```

Such a big station is an easy target, so let's add the primary defense of any space station: a force shield.

## Flattening - `ForceShieldShop`

According to the shop, force shields are delivered in shipments:

```scala
case class ShieldShipment(shields: List[ForceShield])

val shipments: Stream[Nothing, ShieldShipment] = Stream.repeatEffect(
  UIO(ShieldShipment(List.fill(shipmentSize)(ForceShield())))
)
```

They asked for help to define a work flow where they can transform their stream of `shipments` into a stream of `shields`, so they could install them to us. `mapConcat` is the solution here:

> `ZStream#mapConcat` - Maps each element to an iterable, and flattens the iterables into the output of this stream

```scala
val shields: Stream[Nothing, ForceShield] = shipments.mapConcat(_.shields)
```

Now, installing shields is trivial:

```scala
def installShield(stations: DStream): DStream =
  stations
    .zip(shields)
    .map { case (station, shield) => station.installShield(shield) }
```

A space station is not a respectful space station if it doesn't have its own fleet.

## Partitioning - `TieFighterShop`

The requirement comes from our side this time: we need a large amount of tie fighters deployed at once. Let's tell the shop to group our fleet with `grouped` before deploying it:

> `ZStream#grouped` - Partitions the stream with specified chunkSize

Their answer was: "Like shooting [rancors](https://starwars.fandom.com/wiki/Rancor){:target="_blank"} in a cage!"

```scala
class TieFighterShop(groupSize: Int) {
  val tieFighters: Stream[Nothing, TieFighter] = Stream(TieFighter()).forever

  def deployFleet(stations: DStream): DStream =
    stations
      .zip(tieFighters.grouped(groupSize))
      .map { case (station, fleet) => station.deployFleet(fleet) }
}
```

Now we have to deal with the project's proper unknown.

## Changing the environment: `SuperLaserShop`

We probably share the same opinion about the person who came up with this requirement:

![](/assets/images/drevil.jpg){: .align-center}

Besides, this shop seems very suspicious. Apparently, `SuperLaser`s are highly regulated, so the shop has "a guy" who can take anything between 5 to 15 seconds to "make it happen".

The uncertainty means we can't use `DStream` from this point on, we should make the `Random` factor explicit:

```scala
class SuperLaserShop {
  val theGuySchedule = Schedule.randomDelay(5.seconds, 15.seconds)

  val lasers: ZStream[Random with Clock, Nothing, SuperLaser] =
    ZStream.repeatEffectWith(
      UIO(SuperLaser()),
      theGuySchedule
    )

  def installTheLaser(stations: DStream):
    ZStream[Random with Clock, Throwable, UnfinishedSpaceStation] =
      stations
        .zip(lasers)
        .map { case (station, laser) => station.installLaser(laser) }
}
```

I have a bad feeling about this... but sometimes we must let go of our pride and do what is requested of us.

It's not up to us to say the project is complete, so there's one last step in the workflow.

## Filtering - `EmpireAuditor`

Every station has to be inspected by the auditor. When it's approved, a new `DeathStar` can be created, otherwise one can receive a `StationRejected`. Essentially, he will filter and transform the stream.

> `ZStream#collect` - Performs a filter and map in a single step

```scala
object EmpireAuditor {
  case class StationRejected(station: UnfinishedSpaceStation)
    extends IllegalStateException(s"$station does not meet the necessary criteria")
}

class EmpireAuditor {
  def inspect(stations: ZStream[Random with Clock, Throwable, UnfinishedSpaceStation]
             ): ZStream[Random with Clock, Throwable, DeathStar] =
    stations
      .collect {
        case UnfinishedSpaceStation(Some(engine), Some(shield), fleet, Some(laser)) if fleet.nonEmpty =>
          DeathStar(engine, shield, fleet, laser)
        case unfinished =>
          throw StationRejected(unfinished)
      }
}
```

All plans we need are in place, time to do it. As someone once said: "Do. Or do not. There is no try".

## The main workflow

So far, we have **described** workflows. Now we will build the Death Star. With all the descriptions in place, we just need to thread them, take the amount of `DeathStar`s ordered and materialize it. ZStream gives us everything we need to achieve that:

> `ZStream#via` - Threads the stream through the transformation function `f`

> `ZStream#take` - Takes the specified number of elements from this stream

> `ZStream#runCollect` - Runs the stream and collects all of its elements in a list

```scala
class StarshipFactory(
  stationBodyShop: SpaceStationBodyShop,
  propulsionEngineShop: PropulsionEngineShop,
  forceShieldShop: ForceShieldShop,
  tieFighterShop: TieFighterShop,
  superLaserShop: SuperLaserShop,
  auditor: EmpireAuditor) {

  def orderDeathStar(quantity: Int):
    ZIO[Random with Clock, Throwable, List[DeathStar]] =
      stationBodyShop
        .moonShapedSpaceStations
        .via(propulsionEngineShop.installEngine)
        .via(forceShieldShop.installShield)
        .via(tieFighterShop.deployFleet)
        .via(superLaserShop.installTheLaser)
        .via(auditor.inspect)
        .take(quantity)
        .runCollect
}
```

> That’s no moon. It’s a space station.

## Conclusion

Building space stations has never been easier. Using a few `ZStream` operations - `zip`, `map`, `mapConcat`, `group`, `collect`, `via`, `take` and `runCollect` - we managed to build a space station the size of a moon, combining different sources, dealing delays and unexpected random events, easily making them work together.

Have a look at the [repository with the full example](https://github.com/juliano/deathstar-zio-stream){:target="_blank"}.

To learn more about ZIO Streams, I recommend this video, recorded a long time ago, in a galaxy far far away:

[Functional Scala - Modern Data Driven Applications with ZIO Streams by Itamar Ravid](https://youtu.be/bbss7elSfxs){:target="_blank"}

## Post-credits scene

The chief engineer responsible for the project, [a human called Galen Erso](https://starwars.fandom.com/wiki/Galen_Walton_Erso){:target="_blank"}, installed a [thermal exhaust port](https://starwars.fandom.com/wiki/Thermal_exhaust_port){:target="_blank"}, in order to dissipate the excess heat produced by the Hyper Drive Engine. Despite the fact it is not necessary, it seems like no one in the Empire noticed there's now a hole in the space station, that leads directly to the reactor system.

Not that it is an actual problem, I mean, who would think about attacking a giant space station equipped with a super laser?
