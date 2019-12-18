---
layout: single
title: "Functional Scala London 2019: Day 2"
header:
  teaser: /assets/images/london2.jpg
  imgcredit: Image by E. Dichtl from Pixabay
categories:
  - scala
  - opensource
  - community
---

My summary of the second day attending the awesome [Functional Scala London 2019](https://www.functionalscala.com/). If you didn't [read the first part, just do it!](/2019/12/17/functional-scala-london-2019-day-1/)

## Keynote: Modern Data-Driven Applications with ZIO Streams, by [Itamar Ravid](https://twitter.com/iravid_)

Opening the second day, Itamar amazed us with the super powers provided by [ZIO Streams](https://zio.dev/docs/datatypes/datatypes_stream).

{% include figure image_path="assets/images/funscala2019/itamar.jpg" caption="Streams are everywhere" %}

Modern data-driven applications have multi dimensional datasets and the flow of incoming data is non stop. Building an ETL from S3 objects in about 10 minutes, Itamar successfully demonstraded the power of `ZStreams`, tackling the problem with a solution that is resource safe, interruptible and efficient with maybe 80 lines of (very simple) Scala code.

This one is easily between my top 3 of this conference. Besides, Itamar's talk inspired me to build something that I hope, will be sharing here soon :)

## Functional Architecture, by [Piotr Golebiewski](https://github.com/ioleo)

Piotr presented a modular architectural design on top of ZIO to deal with complexity, building small programs and composing them into larger ones.

{% include figure image_path="assets/images/funscala2019/piotr.jpg" caption="Complexity is a growing concern in a software project" %}

I would call his demo a "real world usecase" (`UserService`, `UserStorage`, so on and so far), the sort of example I consider very important; that's exactly what helps selling Scala and its ecossystem.

He showed them how to test the modularized application, organise dependencies, the good and the bad. It was Piotr first presentation, and it was a great one. Well done Pietr!

## ZIO Chunk: Fast, Pure Alternative to Arrays, by [Aleksandra A Holubitska](https://twitter.com/Oleksandra_A)

Another first time speaker! Aleksandra started listing the benefits of using arrays, highlighting the high performance and using benchmarks to compare diferent implementations. Afterwards, she talked about the drawbacks.

{% include figure image_path="assets/images/funscala2019/oleksandra.jpg" caption="Zio Chunk has the benefits of an Array without the downside" %}

She gave us an overview of [ZIO Chunk](https://javadoc.io/doc/dev.zio/zio_2.12/1.0.0-RC12-1/zio/Chunk.html), how to use its modern api, followed by an example using Chunk with [ZIO NIO](https://zio.github.io/zio-nio/).

Congratulations for your great presentation, Aleksandra!

## Caliban: Designing a Functional GraphQL Library, by [Pierre Ricadat](https://twitter.com/ghostdogpr)

Pierre is the creator of [Caliban](https://ghostdogpr.github.io/caliban/), a purely functional library for creating GraphQL backends in Scala. He told us about GraphQL in a nutshell followed by the reasons why the lib was created.

{% include figure image_path="assets/images/funscala2019/ghostdogpr.jpg" caption="Caliban derives the schema from basic types" %}

The talk was a general overview of the lib's capabilities, plus plans for the future. It's the sort of lib important to prove the maturity level of Scala ecosystem.

[Slides available here](https://www.slideshare.net/PierreRicadat/designing-a-functional-graphql-library-204680947)

## Macros and Environmental Effects, by Maxim Schuwalow

A talk about how to eliminate boilerplate code in the ZIO environment, using [ZIO Macros](https://github.com/zio/zio-macros)

{% include figure image_path="assets/images/funscala2019/mix_ab.jpg" caption="One primitive, no magic!" %}

Maxim had some examples with boilerplate using [Doobie](https://tpolecat.github.io/doobie/) and even ZIO, and the differences after performing some local elimination after adopting the macros.

Honestly, I don't have many notes about this talk, which I loved - I was just too busy paying attention!

## Streaming Analytics with Scala and Spark, by [Bas Geerdink](https://twitter.com/bgeerdink)

A presentation about fast data use cases, in different sectors (Finance, Healthcare) and patterns (fraud detection, trend analysis) for instance. Think Big Data, critical domains and real time.

{% include figure image_path="assets/images/funscala2019/bas.jpg" caption="Real world cases of streaming analytics" %}

Bas showed us how to work this data with Spark + Kafka. How to prepare the data, process the streams, look at time windows and how to bound them by watermarks (timestamps).

[Slides available here](https://streaming-analytics.github.io/Styx/presentations/functional-scala.html)

## ZIO Actors, by [Mateusz Sokol](https://twitter.com/mt_sokol)

Mateusz started his talk going back to actors basics: what they are and what they should do. He made a quick demo using [Akka actors](https://doc.akka.io/docs/akka/current/typed/actors.html#akka-actors) and then introduced [ZIO Actors](https://zio.github.io/zio-actors/)

{% include figure image_path="assets/images/funscala2019/sokol.jpg" caption="ZIO Actors are location transparent" %}

Even though ZIO Actors are stateful, they are wrapped in pure effects. Using a State Machine example, Mateusz demoed supervision, location transparency and more. At the very end, he shared the challenges of building the lib.

## Adventures in Type-safe Error Handling, by [Jacob Wang](https://twitter.com/jatcwang)

Jacob went all the way through every abstraction to handle errors, including:

```scala
Either
IO[A] //cats
IO[Either[E, A]]
EitherT[IO, E, A]
Bifunctor IO[+E, +A] //zio
```

What took him to build [Hotpotato](https://jatcwang.github.io/hotpotato/) (best name ever), a type-safe error handling library, based on [Shapeless Coproducts](https://github.com/milessabin/shapeless/wiki/Feature-overview:-shapeless-2.0.0#coproducts-and-discriminated-unions), an interesting alternative.

## Composition using Arrows and Monoidal Categories, by [Oleg Nizhnik](https://twitter.com/Odomontois)

Oleg delivered a proper class about Monoidal Categories Theory. This one took my whole attention, I would suggest you to wait for the video, it was legendary.

{% include figure image_path="assets/images/funscala2019/categories.jpg" caption="Monads are powerful due to functions" %}

[Slides available here](http://slides.com/olegnizhnik/assymondark#/)

## Practical Logic(al) Programming with Dotty, by [Lander Lopez](https://twitter.com/LanderLo)

This presentation was [Dotty](https://dotty.epfl.ch/) in a nutshell. My highlight is about [Union Types](https://dotty.epfl.ch/docs/reference/new-types/union-types.html). Unfortunately at this point I had to take a break, so missed most of the talk.

## Next-Level Type Safety: An Intro to Generalized Algebraic Data Types, by Matthias Berndt

Matthias started describing ADTs, using `Either` as example and talking about how we can reify type information into values. He discussed the limitations regarding some combinators, and introduced Generic ADTs.

{% include figure image_path="assets/images/funscala2019/berndt.jpg" caption="GADTs: many possible generalizations" %}

We watched a case study of designing an expression DSL, first implemented just with regular ADTs and discussing the limitations of this solution. Afterwards, an implementation using GADTs, provinding proper type reification and type check, having a much nicer result.

## Keynote: The Many Faces of Modularity, by [Eric Torreborre](https://twitter.com/etorreborre)

Spoiler: Eric's talk about modularity was awesome. Starting with a comparison between software and Lego, Eric trashed everything that's not modular, and I love this sort of provocation.

{% include figure image_path="assets/images/funscala2019/etorreborre.jpg" caption="Modularity reduces complexity" %}

Eric backed his afirmations about modularity with studies, listed the **many** problems of working with software: critized popular solutions (including ZIO!) in a very respectful way, highlighted that there are topics were modularity is not being properly considered yet (data, for instance), we have weird solutions to problems, we reimplement the same solutions reiventing the ecossytem. It was a long rant! (alert of internal joke)

At the end, he talked about what really matters and how to achieve the best results. Looking back to solutions that were already there, functional programming, encapsulation, types, effects, streams, always seeking for modularity. I am definitely rewatching his talk when available, I have pages and pages full of notes from it!

{% include figure image_path="assets/images/funscala2019/modularity.jpg" %}

That was the final day of the conference. Lots of knowledge, wonderful speakers, community growing. I can't wait for Functional Scala 2020!

[#ScalaThankYou](https://twitter.com/search?q=%23ScalaThankYou) to [Functional Scala 2019](https://twitter.com/FunScala2019) for this conference! The Scala community is lucky to have you!
