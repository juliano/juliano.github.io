---
layout: single
title: "The Journey of an Open Source Developer"
header:
  teaser: /assets/images/hacktoberfest.png
categories:
  - opensource
  - community
  - career
---

[Hacktoberfest](https://hacktoberfest.digitalocean.com/) is an awesome initiative that invites developers from around the world to participate and contribute to Open Source. This is the second year in a row that I managed to beat the challenge, and inspired by it I decided to write a post about how much I learned since I first joined that community.

## A long time ago in a galaxy far, far away...

The year was 2010, where the universe made his move to make me cross paths with my good friend [Jonas Abreu](https://twitter.com/jonasabreu). Jonas is the creator of [Mirror](http://projetos.vidageek.net/mirror/mirror/), a simple DSL layer over Java Reflection API, making meta programming easier. There was a feature request for a proxy creation capability open, and Jonas asked me if I was interested in implementing it. I accepted the challenge.

I suffered for three days, dealing with code that I didn't even understand... until I did. And I still remember the feeling that my triumph brought me that day. That pain was the knowledge getting inside my brain!

My very first contribution to open source. Here's what I learned contributing to **Mirror**:

- Lots about Java meta programming
- Proxy classes, [Javassist](https://www.javassist.org/) and [cglib](https://github.com/cglib/cglib)
- Receiving and providing useful/respectful feedback about code
- Designing a tool that will be used by others
- Some interesting [dark magic](https://github.com/vidageek/mirror/blob/master/src/main/java/net/vidageek/mirror/provider/java/ObjenesisConstructorBypassingReflectionProvider.java)

> I loved the idea of contributing to the greater good, while challenging myself and working with brilliant engineers, way smarter than me.

![](/assets/images/scala-logo.png){: .align-right}I started taking part community initiatives, like running coding dojos. Doing that I met more great devs and we started a small group. At some point, that group decided to learn Scala. We were studying, coding together and eventually started [Scaladores](https://www.meetup.com/scaladores/), the Scala user group of São Paulo.

Eventually, Jonas started talking about a different approach for learning he had been studying, called [deliberate practice](https://jamesclear.com/deliberate-practice-theory).

> What if we could put all these recently acquired knowledge altogether?

## Gamification + Deliberate Practice = Aprenda

The result was [Aprenda](https://aprenda.vidageek.net/), a learning platform that mix gamification and deliberate practice, making it easier to learn html, regex or git.

<figure class="align-right">
  <img src="{{ 'assets/images/aprenda.png' | absolute_url }}" >
  <figcaption>What do you want to learn today?</figcaption>
</figure>

This is what I learned working on Aprenda:
- Deliberate Practice
- Gamification
- Design a system using Actors
- Parser Combinators

Entirely different topics this time, technically and conceptually. However I couldn't dedicate a good amount of time to this project. I had other problems to deal with, what brew some new ideas.

I had finished working in a Ruby on Rails project, and moved to a team working with Java and Spring. Many CRUDs, a lot of repetition, what gave me an idea. So I started a new project.

## Writing my own Gem

![](/assets/images/ruby-logo.png){: .align-right}The idea was clear, I wanted to generate that similar code the same way one can do with `rails g scaffold`. Using [Thor](http://whatisthor.com/), the same gem powering Rails generators, I created [Spring MVC Scaffold](https://github.com/juliano/springmvc-scaffold). Now, everything I needed to do to create a CRUD was:

```shell
springmvc scaffold product name:string value:double active:boolean
```

Even though it's not being maintained anymore, it's still available in [Rubygems](https://rubygems.org/gems/springmvc-scaffold). Here are my takeaways from my first public tool:

- Generating code/files with Thor
- Defining commands for a cli
- Organising code of a ruby lib
- Creating a ruby gem
- [Travis](https://travis-ci.com/) for opensource
- Publishing to Rubygems

> After solving a common problem, think about making that solution available. Most likely, other people have similar problems.

And I tell you what, after that project, I went to work with a tech environment full of problems.

## Working with Microsoft Tech

![](/assets/images/dotnet-logo.png){: .align-left}<br>While working with Microsoft .NET, I found a few issues that other communities had already solved. My new opportunity to contribute was a matter of porting those solutions.

### [Selenia](https://github.com/juliano/Selenia/)

In my opinion, [Selenium](https://www.seleniumhq.org/) API has always been pretty bad. Selenia is a DSL to write concise UI tests in C#, so instead of having:

```c#
IWebDriver driver;
ChromeOptions options = new ChromeOptions();
options.addExtensions(new File("/path/to/extension.crx"));
ChromeDriver driver = new ChromeDriver(options);
driver = new ChromeDriver();
driver.Navigate().GoToUrl("http://www.google.com/ncr");

IWebElement query = driver.FindElement(By.Name("q"));
query.SendKeys("Selenium");
query.Submit();

driver.Quit();
```

We can write

```c#
Open("http://www.google.com/");
S(By.Name("q").Value("Selenia").Enter();
```

If you are wondering, the answer is no, it's not necessary to close the driver yourself.

### [CSharp.Fun](https://github.com/brenoferreira/CSharp.Fun/)

When building reports, it's common to navigate the hierarchy of objects, and even more common is *to check if the current node is not null before the next step*. My friend [Breno](https://twitter.com/breno_ferreira) and I implemented a few Monadic Types to deal with that pain, like [Option](https://en.wikipedia.org/wiki/Option_type) and Try:

```c#
// Option
var option = Option.From<string>(null);

option.Should().Be(Option.None);

// Try
var failure = Try.From<int>(() => { throw new Exception(); });
var success = failure.Recover((Exception ex) => 1);

success.Value.Should().Be(1);
```

### [Ioget](https://github.com/juliano/Ioget/)

Immutability makes a developer's life easy. Back in the day, asp.net mvc would disagree, the only way to instantiate objects was via setters. I had to do something about it.

I built Ioget to to help with unmarshalling request parameters in Web applications. HTTP request parameters are strings, so Ioget looks for the best way to parse those paams according to the given class, instantiating objects via their constructors, therefore making them immutable.

Even though I published it to [nuget](https://www.nuget.org/packages/Ioget/), I've never managed to integrate it with asp.net, at the time I stoped working with Microsoft technology. Here are my learnings from the three projects:

- [Reification](https://en.wikipedia.org/wiki/Reification_(computer_science))
- Internals of asp.net mvc
- Implementing monads
- Internals of .net framework

Regarding internals, I really like this snippet, used in Selenia to close the driver:

```c#
private void MarkForAutoClose(IWebDriver driver) =>
  AppDomain.CurrentDomain.DomainUnload += (s, e) => driver.Quit();
```
<br>

## There and Back Again

I moved to London in 2016, what paused my contributions for a while. Eventually, I watched a talk from [Gustavo Amigo](https://github.com/gustavoamigo) about his project [quill-pgsql](https://github.com/gustavoamigo/quill-pgsql), an extension to support Postgres data types with [Quill](https://getquill.io/). He mentioned that the project was in its early moments, ideal for someone to join. And was missing writing some Scala.

After a few pull requests, I decided to contribute with the main project.
<a href="https://getquill.io/">
![](/assets/images/quill-logo.png){: .align-center}
</a>

Quill is a Compile-time Language Integrated Queries for Scala. I consider it the most challenging (and interesting) project I have ever contributed to. It's been a few years since I started, and nowadays I am [one of the maintainers](https://github.com/getquill/quill). Here's what I learned working on Quill:

- Abstract Syntactic Trees
- The Dark Arts also known as Scala Macros
- How to make code extensible via implicits
- Exclusive/weird SQL rules

The next project is strongly related. We have a module `quill-async`, which uses [an async db driver](https://github.com/mauricio/postgresql-async) that is not being being maintained anymore. Quill's creator [Flavio Brasil](https://twitter.com/flaviowbrasil) suggested we could write a new async driver. That's how NDBC started.

[Non-Blocking Database Connectivity (NDBC)](https://ndbc.io/) is a fully async alternative to JDBC. It's architecture was designed to provide a high performance, non-blocking connectivity to the database on top of [Trane.io Futures](http://trane.io/) and [Netty 4](https://netty.io/).


```java
// Create a Config with an Embedded Postgres
Config config = Config.create("io.trane.ndbc.postgres.netty4.DataSourceSupplier", "localhost", 0, "user")
                      .database("test_schema")
                      .password("test")
                      .embedded("io.trane.ndbc.postgres.embedded.EmbeddedSupplier");

// Create a DataSource
DataSource<PreparedStatement, Row> ds = DataSource.fromConfig(config);

// Define a timeout
Duration timeout = Duration.ofSeconds(10);

// Send a query to the db defining a timeout and receiving back a List
List<Row> rows = ds.query("SELECT 1 AS value").get(timeout);

// iterate over awesome strongly typed rows
rows.forEach(row -> System.out.println(row.getLong("value")));
```

More knowledge acquired:

- Binary protocols
- Netty 4
- Weaknesses of the Java type system
- Fully implementing functional structures

That's where I am at the moment. I share my attention between Quill and Ndbc, trying to make them work together.

## Conclusion

This almost 10 years open source journey taught me something extremely valuable:

> Open Source is about sharing knowledge. When I am solving an issue I am acquiring knowledge. When I send a pull request, I am spreading that knowledge. Knowledge brings more knowledge!

I strongly recommend you to become part of the open source community. It can be difficult, specially in the beginning, but everything you will learn will make you a better developer. I promise!

## Plans for the future

The contributions I have in mind for the near future are:
- [Implementing array support in Finagle Postgres](https://github.com/finagle/finagle-postgres/issues/55)
- Clojure wrapper for NDBC
- `ndbc-spring` module
- cli in Rust

Any suggestion? Interesting ideas? Let me know in the comments!
