---
title: "EIP and Apache Camel"
date: 2020-03-31T13:06:37+08:00
allowComments: true
draft: true
---

[Apache Camel][camel] is THE Java implementation of
[Enterprise Integration Patterns (EIP)][eip] that eases integrating
applications within the organization. Until recently, I realized that
not reading the EIP book is doing a disservice to myself when attempting
to use Apache Camel.[^1]

Even though EIP focuses on using messaging for integration, the following
is the complete list of integration styles stated in the book:

* File transfer: source application outputs a file with pre-agreed format
  to a pre-agreed destination that the integrator can pick up and process.
  Changes within applications won't affect one another, as long as the
  output file sticks to the pre-agreed format. The main disadvantage is
  the low frequency in outputting the file: although outputting the as
  frequent as necessary is possible, the challenge still lies in ensuring
  the outputs are processed and not lost.
* Shared database: different applications write to a common database to
  share information, thus overcoming the low frequency limitation when
  using file transfers. The main disadvantage lies in getting different
  applications to agree to the same schema the shared database demands,
  especially when applications are vendor software that can evolve their
  application's schema as the vendor deems fit.
* Remote procedure invocation: aka Remote Procedure Call (RPC), applications
  expose APIs to allow others to invoke, thus allowing applications to
  still encapsulate internal representations. The main disadvantage is that
  applications could be tightly coupled together because invoking
  applications may need to know the sequence in which they should invoke
  the remote calls. RPIs may also lead to slow and unreliable system
  if invoking systems aren't aware of the difference between remote and
  local calls.
* Messaging: source application outputs small data packets more frequently
  and asynchronously than file transfers and receiving applications processes
  them as soon as the data packets reach them. Unlike file transfers,
  messages can be transformed while "in-flight" and the source/receiving
  applications won't even need to know of such transformations. Such
  transformation behavior means the glue code in the integration layer
  are more involved, as compared to the other integration styles.

## High-level Components

The following are the high-level components of a messaging system:

* Message: a data packet the source application outputs and the receiving
  application processes; note that the message may be transformed within
  the messaging system without letting the source and receiving applications
  know
* Message channel: a "pipe" that the sender writes to and receiver reads from
  messages in the messaging system
* Pipes and filters: an architecture that chains multiple processing
  components with channels
* Message router: a component that directs messages in the messaging system
  to the relevant component or the receiving application depending on
  the message's content, e.g., the router may route the invoice to an
  error queue when the billing address is wrong
* Message translator: a conversion of the message, possibly from one format
  to another so that the sender and receiver can use the format that
  are suitable for them
* Message endpoint: the entry/exit point of sender/receiver to the messaging system
  that knows know the application and the messaging system work

Each component has its own set of design patterns which Apache Camel
has implemented.

## Message

Senders can send messages synchronously (request-response) or
asynchronously (fire-and-forget). Camel differentiates the two in the
message container: [`org.apache.camel.Exchange`][exchange]. `Exchange`
has a `pattern` property that indicates whether the exchange type should
be an `InOut`, `InOnly`, or `InOptionalOut` (enum values from
[`org.apache.camel.ExchangePattern`][exchangePattern].

## Setting up Apache Camel

`build.gradle.kts` looks as follows:

```kotlin {linenos=table,  hl_lines=[15,17,18,19,20]}
plugins {
    id("org.jetbrains.kotlin.jvm") version "1.3.72"
    application
}

repositories {
    mavenLocal()
    mavenCentral()
}

dependencies {
    implementation(platform("org.jetbrains.kotlin:kotlin-bom"))
    implementation("org.jetbrains.kotlin:kotlin-stdlib-jdk8")

    implementation("org.slf4j:slf4j-nop:1.7.30")

    for (n in listOf("core-engine", "core-languages", "bean", "direct",
            "stream")) {
        implementation("org.apache.camel:camel-$n:3.3.0")
    }

    implementation("org.jetbrains.kotlin:kotlin-test")
    implementation("org.jetbrains.kotlin:kotlin-test-junit")
}

application {
    mainClassName = "eip.Appkt"
}
```

Unlike 2.x series, 3.x series have broken up into multiple smaller jars
for finer control on what's being pulled in for use:

* `core-engine`: without which, nothing works
* `core-languages`: for using scripting languages in building routes with
  Camel's DSL

[^1]: [Camel in Action, 2nd Edition][cia2e] actually did recommend
      reading the EIP book shortly into chapter 1, but I ignored that
      advice :sweat_smile:

[camel]: https://camel.apache.org
[eip]: https://www.enterpriseintegrationpatterns.com
[cia2e]: https://www.manning.com/books/camel-in-action-second-edition
[exchange]: https://www.javadoc.io/doc/org.apache.camel/camel-api/3.3.0/org/apache/camel/Exchange.html
[exchangePattern]: https://www.javadoc.io/doc/org.apache.camel/camel-api/3.3.0/org/apache/camel/ExchangePattern.html
