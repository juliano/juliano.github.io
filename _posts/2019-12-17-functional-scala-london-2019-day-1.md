---
layout: single
title: "Functional Scala London 2019: Day 1"
header:
  teaser: /assets/images/london1.jpg
  imgcredit: Photo by Aron Van de Pol on Unsplash
categories:
  - scala
  - opensource
  - community
---

Last week I attended the [Functional Scala London 2019](https://www.functionalscala.com/), a conference with an incredible lineup of prominent figures of the Scala world. Organized by [John De Goes](https://twitter.com/jdegoes) and [Aleksandra A Holubitska](https://twitter.com/Oleksandra_A), this conference is a proper contribution to the community: 2 days of knowledge sharing at no cost for the attendees. This is my summary of every talk presented at day 1.

## Keynote: Against noise by Paul Phillips

Paul couldn't be present, so we watched a video he recorded for the conference. He cracked some funny jokes (being invisible) before starting his point, using [`df`](http://linuxcommand.org/lc3_man_pages/df1.html) as example: it has a crappy output. In order to extract useful information using `df`, some options are necessary, whereas the default case should be the one providing an output that's easy to understand.

`df` is noisy, and visual noise is worse than audio noise. That was his motivation to start his new project: `xs`

{% include figure image_path="assets/images/funscala2019/paul_phillips.jpg" caption="I want to work in peace without being assaulted all the time by meaninless garbage" %}

Paul is a great presenter. Having him was a nice start to the conference!

## Introduction to Interruption by [Jakub Kozlowski](https://twitter.com/kubukoz)

A talk about functional effects and how they don't break referential transparency, being purely functional. Jakub showed us an "Interruption story" example, and discussed what are the options to code it, mentioning Futures (no go), Fibers (easy to mess up) and finally Effects.

{% include figure image_path="assets/images/funscala2019/kubukoz.jpg" caption="Pretty much everything can be interrupted nowadays" %}

Some takeaways from his talk:

- Avoid concurrency
- If you are using it, avoid start/fork
- Use built-in [parallel operators](https://typelevel.org/cats-effect/datatypes/io.html#parallelism)
- Use [Deferred](https://typelevel.org/cats-effect/concurrency/deferred.html)
- Small, high level, compositional abstractions
- Keep the concurrency details away from domain
- Trust the laws, nothing else

[Slides available here](https://speakerdeck.com/kubukoz/introduction-to-interruption)

## Making Algorthms work with Functional Scala by [Karl Brodowsky](https://twitter.com/bk1_168)

Karl's talk was heavily focused on algorithms and how we should be aware that, sometimes, when being purely focused on one aspect, we can forget about other important aspects of our software, like performance. Using sorting algorithms as examples, he measured and presented how immutability can be expensive.

{% include figure image_path="assets/images/funscala2019/brodowski.jpg" caption="Use mutability internally - don't allow it to leak" %}

He finished talking about more performatic sorting algorithms, like [Flash Sort](https://brodowsky.it-sky.net/2019/03/19/flashsort-in-scala/).

## Solving the Scala Notebook Experience by [Jeremy Smith](https://twitter.com/jeremyrsmith) & [Jonathan Indig](https://twitter.com/indigjonathan)

This talk introduced [Polynote](https://polynote.org/), a polyglot notebook environment. They started discussing the pain points of working with Scala + Spark in a notebook, what was the motivation to build Polynote.

In summary, they walked-through the process used to built the tool... by one developer!

{% include figure image_path="assets/images/funscala2019/polynote.jpg" caption="The Scala community made the project possible" %}

> You don't need to be an expert on category theory to be productive with FP.

> Don't `F[_]` around

I don't work with this sort of software, but I had some brief contact with [Jupyter](https://jupyter.org/) in the past. Polynote looks like a modern and superior tool for me, with an interesting set of features.

[Slides available here](https://jeremyrsmith.github.io/polynote-2019-slides/#1)

## Mixing Scala & Kotlin by [Alexey Soshin](https://twitter.com/alexey_soshin)

Alexey talked about Scala adoption, how one can educate developers, fire them or provide an environment with polyglot microservices.

{% include figure image_path="assets/images/funscala2019/soshin.jpg" caption="Let them do Java, or Kotlin (in Scala)" %}

Some of the highlights about Scala and Kotlin interop were the differences between Companion Objects and Functions. Sometimes, it's necessary to use the common ground between them: Java. His example was about Futures, from Kotlin to Java `CompletableFutures` and them to Scala.

## Prototyping the Future with Functional Scala, by [Mike Kotsur](https://twitter.com/s_fcopy)

Mike talked about how good the Scala ecosystem is for prototyping. He told us about a project that speeds up Docker containers, innitialy prototyped in Python and Django.

{% include figure image_path="assets/images/funscala2019/kotsur.jpg" caption="Don't pitch FP or Scala, pitch a solution" %}

I would say his presentation was an use case of architectural change, mostly adopting [cats effect](https://typelevel.org/cats-effect/typeclasses/effect.html).

## Unveiling ZIO Test, by [Adam Fraser](https://twitter.com/adamfraser)

This was one of my favourite talks. Adam introduced [ZIO Test](https://zio.dev/docs/usecases/usecases_testing), and honestly, it seems awesome, I will give it a try as soon as I can. It implements Tests as values, provides an easy syntax, generators and property based tests out of the box.

{% include figure image_path="assets/images/funscala2019/fraser.jpg" caption="Tests frameworks need to evolve" %}

I strongly suggest you to [have a look at the documentation](https://zio.dev/docs/overview/overview_testing_effects) and how to [get started with ZIO Test](https://medium.com/@wiemzin/get-started-with-zio-test-7a27da355498).

[Slides available here](https://github.com/adamgfraser/unveiling-zio-test/blob/master/unveiling-zio-test.pdf)

## Let's Gossip! by [Dejan Mijic](https://twitter.com/dejan_mijic) and [Przemyslaw Wierzbicki](https://twitter.com/pshemass)

In summary, a talk about [ZIO Keeper](https://zio.github.io/zio-keeper/), a purely-functional, type-safe library for building distributed systems.

> Distributed systems are everywhere

The main characteristics are:

- Composable and easily replacable building blocks
- Resilient (failure friendly)
- Strong typed and resource-safe
- Security as an opt-in feature

## Ray Tracing with ZIO, by [Pierangelo Cecchetto](https://twitter.com/pierangelocecc)

Pierangelo presented a [pure functional, modular ray tracer library he built on top of ZIO](https://github.com/pierangeloc/ray-tracer-zio). I understand it as an abstraction behind the behaviour of rays, reflection and visual effects.

{% include figure image_path="assets/images/funscala2019/cecchetto.jpg" caption="Implementing World reflection" %}

[Slides availabe here](https://www.slideshare.net/PierangeloCecchetto/ray-tracing-with-zio)

## Invertible Programs, by [Sergei Shabanau](https://twitter.com/SShabanau)

In this talk, Sergei discussed invertible programs. Think about inversions between strings and ints, or bytes and a http request. Defining a type-safe specification of a grammar using [parser combinators](https://en.wikipedia.org/wiki/Parser_combinator), Sergei found an interesting solution in [Parserz](https://github.com/spartanz/parserz).

> Complex problems do not necessarily require complex solutions

The killer feature of Parserz when compared to other libs is simple: it is easy to go both sides.

## Hyper-pragmatic Pure FP Testing with DIStage-Testkit, by [Pavel Shirshov](https://twitter.com/shirshovp) and [Kai](https://twitter.com/kai_nyasha)

The talk started with a discussion: which tests are good and which are bad? Bad tests are slow, unstable, don't survive refactoring, hard to maintain. In summary, bad tests are expensive.

{% include figure image_path="assets/images/funscala2019/7mind.jpg" caption="Some tests prove themselves useful and some do not" %}

Pavel and Kai suggested a different terminology, instead of unit/functional/integration, two categories for encapsulation Blackbox and Whitebox tests. Then, three levels of isolation: Atomic, Group or Communication tests.

 Based on these initial premises, they presented a testcase they build with [`distage-testkit`](https://izumi.7mind.io/latest/release/doc/distage/distage-testkit.html) which simplifies the test environment setup, provinding tests dependencies in a smart and non-invasive approach. The way it provides Docker containers I found particularly interesting.

[Slides available here](https://github.com/7mind/slides/blob/master/07-distage-tests-functional-scala/distage-tests-functional-scala.pdf)

## Keynote: Unleash the Fury, by [Jon Pretty](https://twitter.com/propensive)

The Keynote closing the first day was delivered by Jon Pretty, a discussion about the top 10 problems with Scala and what we, as a community, can do to solve them. He talked about the news for 2020, and the expected impact. He talked about his current project, [Fury](https://fury.build/), a dependency manager and build tool for Scala and (potentially) other JVM languages. The tool has a friendly user interface and tries to solve serious problems like publishing libs.

{% include figure image_path="assets/images/funscala2019/pretty.jpg" caption="Scala is awesome" %}

Great message to close the first day :)

And this was the first day of the conference in a nutshell. A considerably amount of knowledge for one day, but was totally worth it. I am very thankful for this conference, from the community to the community! A Big [#ScalaThankYou](https://twitter.com/search?q=%23ScalaThankYou) to everyone that made [Functional Scala 2019](https://twitter.com/FunScala2019) happen!
