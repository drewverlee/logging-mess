---
title: "Logging in Clojure: Making Sense of the Mess"
uri-slug: "logging-in-clojure-making-sense-of-the-mess"
link: "https://lambdaisland.com/blog/2020-06-12-logging-in-clojure-making-sense-of-the-mess"
edit: "https://lambdaisland.com/blog/2020-06-12-logging-in-clojure-making-sense-of-the-mess/edit"
created-on: "2020-06-12T10:10:45+00:00"
updated-on: "2020-06-12T16:55:10+00:00"
---

[![](https://img.lambdaisland.com/d2d44e2500e32e9d68ad8ca2e5a6880e.png){:style "max-width: 900px"}](https://img.lambdaisland.com/d2d44e2500e32e9d68ad8ca2e5a6880e.png)

You may also like [Dates in Clojure: Making Sense of the
Mess](https://lambdaisland.com/blog/2017-07-26-dates-in-clojure-making-sense-of-the-mess)

Logging seems like a simple enough concept. It's basically `println` with a
couple of extra smarts. And yet it can be oh so confounding. Even when you
literally do not care a sigle bit about logging you may still be sent down a
rabbit hole because of warnings of things that are or aren't on the classpath.
What even is a classpath?

## The Logging Wars

Most of the mess that is logging we inherit from Java. During the first half of
the nillies logging libraries and frameworks were fighting for dominance in
what I will dramatically dub "The Logging Wars". Around 2006 things started
stabilizing, but the scars from this period are still visible in our dependency
graphs today.

Here's a typical example, snippets like this you find in nearly every production
Clojure project. To make sense of it we need to dive into a little bit of
history

``` clojure
{:deps
 {,,,
  org.slf4j/slf4j-api {:mvn/version "1.7.30"}
  org.slf4j/jul-to-slf4j {:mvn/version "1.7.30"}
  org.slf4j/jcl-over-slf4j {:mvn/version "1.7.30"}
  org.slf4j/log4j-over-slf4j {:mvn/version "1.7.30"}
  org.slf4j/osgi-over-slf4j {:mvn/version "1.7.30"}
  ch.qos.logback/logback-classic {:mvn/version "1.2.3"}
  }}
```

The first logging library to gain traction in the Java world was Log4j. Log4j
1.0 was released in January 2000, but its history goes back to 1996, when it was
developed as part of an EU research project called
[SEMPER](http://www.semper.org/).

Log4j is credited with being the first to introduce the hierarchical loggers, a
concept that would be adopted by nearly every logging library that came after
it, not just in Java.

Meanwhile Sun Microsystems was working on including logging functionality in the
JDK, [JSR-47](https://www.jcp.org/en/jsr/detail?id=47) was initiated in 1999,
and the java.util.logging (JUL) package was included in Java 1.4 in February 2002. The
API has a striking similarity with Log4j.

## Facade/off

Because it had taken Java so long to include an official logging API a bunch of
different libraries had sprung up to fill the void. This was kind of annoying,
because third party libraries would all depend on different logging
implementations, making it hard to get a single unified log of what was going on
in your application. Application developers also became more prudent about
coupling their code to a specific logging implementation, given how things were
shifting.

And thus the idea of a "Logging Facade" was born. A Logging Facade provides an
intermediate interface. Application or library code can use whatever logging
library code they prefer, the logging calls simply get forwarded to the facade.

The facade in turn will forward these to a specific logging implementation, so
you could swap out Log4j for java.util.logging for Avalon Logkit without having
to change a line of code.

The first library to popularize this idea was Jakarta Commons Logging (JCL),
which hit 1.0 in 2002, right around the time that Java 1.4 was released. JCL
would later be renamed to Apache Commons Logging.

Apparently JCL had some shortcomings, and so in 2005 SLF4J was conceived, the
Simple Logging Facade for Java. SLF4J, like JCL, is able to output to various
pre-existing libraries, including Log4j and java.util.logging, but the SLF4J
author also wrote a logging backend that is "native" to SLF4J, Logback.

SLF4J and Logback are both written by Ceki Gulcu who was already involved way
back with the original Log4j, so supposedly he's had some time to mull over the
problem space, and you could say that SLF4J is the de facto standard now.

Given that SLF4J has become somewhat of a standard in the Java world, and that
you can pipe virtually every logging library out there into SLF4J, I would
recommend to center the logging of your application around SLF4J. Add the
necessary adapters to forward all other logging libraries to SLF4J. For the
backend I recommend Logback. That's how you end up with the snippet at the top.

## Logback Mountain

SLF4J will automatically figure out which backend is available, and use that
one. This means that to make this work reliably you need to make sure there is
one and only one logging backend in your dependencies. A lot of libraries will
indirectly pull in the "nop" backend which is simply a placeholder that doesn't
log anything. Use `lein deps :tree` or `clj -Stree` and add exclusions where
necessary. For example:

``` clojure
org.clojure/tools.deps.alpha {:mvn/version "0.6.480", :exclusions [org.slf4j/slf4j-nop]}
```

Some of the libraries that you want to pipe into SLF4J will do their own
auto-detection of where their logs will go, leading to more stuff to exclude.
You probably want to avoid having any of these in your list of dependencies,
directly or indirectly

- commons-logging
- log4j
- org.apache.logging.log4j/log4j
- org.slf4j/simple
- org.slf4j/slf4j-jcl
- org.slf4j/slf4j-nop
- org.slf4j/slf4j-log4j12
- org.slf4j/slf4j-log4j13

You can try this snippet to see if you need to add any extra exclusions.

``` shell
clj -Stree | egrep '(commons-logging.*|log4j.*|org\.apache\.logging\.log4j/log4j|org\.slf4j/simple|org\.slf4j/slf4j-jcl|org\.slf4j/slf4j-nop|org\.slf4j/slf4j-log4j12|org\.slf4j/slf4j-log4j13) '
```

or for Leiningen:

``` shell
lein deps :tree | egrep '(commons-logging.*|log4j.*|org\.apache\.logging\.log4j/log4j|org\.slf4j/simple|org\.slf4j/slf4j-jcl|org\.slf4j/slf4j-nop|org\.slf4j/slf4j-log4j12|org\.slf4j/slf4j-log4j13) '
```

If you are using Leiningen then it's possible to declare exclusions at the top
level and have them apply to all dependencies, which is quite useful. Have a
look at Stuart Sierra's [log.dev](https://github.com/stuartsierra/log.dev/blob/master/project.clj)
example project for what that looks like.

To configure Logback all you need is a `logback.xml` file, which you typically
put under `resources/`. Do make sure `"resources"` is part of your `:paths`
(deps.edn). Leiningen adds it to its `:resource-paths` automatically.

This snippet is a good starting point:

```xml
<configuration scan="true" scanPeriod="5 seconds">
  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
      <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
    </encoder>
  </appender>
  <root level="INFO">
    <appender-ref ref="STDOUT" />
  </root>
  <logger name="org.apache.kafka" level="WARN" />
</configuration>
```

- It reloads `logback.xml` every five seconds, so it's easy to change the
  configuration without having to restart your REPL
- It simply outputs to STDOUT, which is great during development, but also very
  common for production app, since whatever cloud platform you are using can
  aggregate the logs from there.
- It sets the log level of the root logger to 'INFO', so by default you'll see
  errors, warnings, some information output about things like configuration, but
  it won't be too noisy.
- Then you can bump the level up or down for specific namespaces/loggers.

## There's Something About Clojure

Note that so far I haven't mentioned anything yet about Clojure libraries. But
given the above all you need is a pretty lightweight library that outputs to
SLF4J. Ideally said library would be able to log Clojure data structures
directly, so you can reap the benefits of structured logging (logs as data).
Which brings me to my favorite Clojure logging library
[io.pedestal.log](http://pedestal.io/api/pedestal.log/io.pedestal.log.html).

There are various other Clojure libraries, some of them have appealing
propositions, and you may or may not decide to adopt them in your projects. I
hope that with the background this article provides you can now more easily
assess where they fit in the ecosystem, and what they bring to the table.

You may also have come across
[clojure.tools.logging](https://github.com/clojure/tools.logging). This is an
attempt to provide a logging facade directly in Clojure. It has support for
various backends, including SLF4J. It provided an idiomatic API for logging from
Clojure early on (2011), without dictating a specific backend. Nowadays I think
there are better alternatives.

[Unilog](https://github.com/pyr/unilog) provides a more Clojure-esque
configuration interface to Logback, allowing you to use EDN files instead of
`logback.xml`.

Some library authors decided they had enough of this mess, and started over from
scratch, implementing an entire logging stack in Clojure. These include
[Timbre](https://github.com/ptaoussanis/timbre) and
[μ/log](https://github.com/BrunoBonacci/mulog). Timbre is particularly
impressive in that it can still act as a backend for SLF4J.

μ/log pushes the idea of structured logging in a distributed context, where logs
are just another data stream akin to a Kafka topic, and indeed it can log to
Kafka directly, as well as ElasticSearch, Kinesis, or plain files or stdout.

## Revenge of the Log4j

After a small decade of stability the Java folks decided to stir things up again
by coming up with Log4j 2. This is really a grand rewrite, and so it's
unfortunate they didn't just give it a new name.

Log4j 2 [tries to learn](https://logging.apache.org/log4j/2.x/) from all that
came before it. It separates logging from implementation, is said to have better
performance in multi-threaded applications, has first class support for
data-based log messages (rather than strings), the list goes on.

I think Log4j 2 holds some interesting potential for Clojure, since it's well
connected to the rest of the ecosystem (adapters to and from other logging
libraries and facades), and has support for `java.util.Map` based messages (an
interface implemented by Clojure's maps). I'd be curious to see Clojure
libraries explore what they can do with this.

_Update:_ Sean Corfield pointed out that they are using clojure.tools.logging with Log4j 2 as the backend, which seems like a good way to get the benefits of Log4j 2 today. Certainly an option to consider!

## Conclusion

SLF4J + Logback has been my default for some time, and one that I've seen used
with success in real world projects, in particular in combination with
pedestal.log. It's an approach that really leans in to Clojure's philosophy of
making good use of its host instead of reinventing the wheel, and Logback's
reload functionality make it a great fit for a REPL driven workflow.

Timbre however does form a strong contender, particularly for people who prefer
a pure Clojure approach, and with its SLF4J bridge it can still play the same
role as Logback, thus unifying application and library logging.

This post has been all about Clojure, luckily on ClojureScript things are a
little less messy. The Google Closure library contains `goog.log`, which offers
an API inspired by Log4j, and [Glögi](https://github.com/lambdaisland/glogi)
offers an idiomatic ClojureScript interface inspired by pedestal.log.

Happy logging!
