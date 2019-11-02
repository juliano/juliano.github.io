---
layout: single
title: "The Journey of an Open Source Developer"
header:
  teaser: /assets/images/hacktoberfest.png
  imgcredit: "Photo by Patrick Tomasso on Unsplash"
categories:
  - opensource
  - career
---

[Hacktoberfest](https://hacktoberfest.digitalocean.com/) is an awesome initiative that invites developers from around the world to participate and contribute to Open Source. This is the second year in a row that I managed to beat the challenge, and inspired by it I decided to write a post about how much I learned since I first joined that community.

# A long time ago in a galaxy far, far away...

The year was 2010, where the universe made his move to make me cross paths with my good friend [Jonas Abreu](https://twitter.com/jonasabreu). Jonas is the creator of [Mirror](http://projetos.vidageek.net/mirror/mirror/), a simple DSL layer over Java Reflection API, making meta programming easier. There was a feature request for a proxy creation capability open, and Jonas asked me if I was interested in implementing it. I accepted the challenge.

I suffered for three days, dealing with code that I didn't even understand... until I did. And I still remember the feeling that my triumph brought me that day. That pain was the knowledge getting inside my brain!

My very first contribution to open source. Here's what I learned contributing to **Mirror**:

- Lots about Java meta programming
- Proxy classes, [Javassist](https://www.javassist.org/) and [cglib](https://github.com/cglib/cglib)
- Receiving and providing useful/respectful feedback about code
- Designing a tool that will be used by others
- Some interesting [dark magic](https://github.com/vidageek/mirror/blob/master/src/main/java/net/vidageek/mirror/provider/java/ObjenesisConstructorBypassingReflectionProvider.java)

> I loved the idea of contributing to the greather good, while challenging myself and working with brilliant engineers, way smarter than me.

![image-right](/assets/images/scala-logo.png){: .align-right}I started taking part community initiatives, like running coding dojos. Doing that I met more great devs and we started a small group. At some point, that group dceided to learn Scala. We where studying, coding together and eventually started [Scaladores](https://www.meetup.com/scaladores/), the Scala user group of São Paulo.

Eventually, Jonas started talking about a different approach for learning he had been studying, called [deliberate practice](https://jamesclear.com/deliberate-practice-theory).

> What if we could put all these recently acquired knowledge altogether?

# Gamification + Deliberate Practice = Aprenda

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

# Writing my own Gem

![](/assets/images/ruby-logo.png){: .align-right}The idea was clear, I wanted to generate that similar code the same way one can do with `rails g scaffold`. Using [Thor](http://whatisthor.com/), the same gem powering Rails generators, I created [Spring MVC Scaffold](https://github.com/juliano/springmvc-scaffold). Now, everything I needed to do to create a CRUD was:

```shell
springmvc scaffold product name:string value:double active:boolean
```

Even though it's not being maintained anymore, it's still in [Rubygems](https://rubygems.org/gems/springmvc-scaffold). Here are my takeaways from my first public tool:

- File generation with Thor
- Defining commands for a cli
- Organising code for a ruby gem
- [Travis](https://travis-ci.com/) for opensource
- Publishing to Rubygems

