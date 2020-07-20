---
title: "EIP and Apache Camel"
date: 2020-03-31T13:06:37+08:00
allowComments: true
draft: true
tags: [kotlin, apache camel]
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

* [Message](#messages): a data packet the source application outputs
  and the receiving application processes; note that the message may
  be transformed within the messaging system without letting the source
  and receiving applications know
* [Message channel](#message-channels): a "pipe" that the sender writes
  to and receiver reads from messages in the messaging system
* Pipes and filters: an architecture that chains multiple processing
  components with channels; large processing tasks could be broken down into
  multiple smaller, independent steps (filters) that are connected by
  channels (pipes)
* Message router: a component that directs messages in the messaging system
  to the relevant component or the receiving application depending on
  the message's content, e.g., the router may route the invoice to an
  error queue when the billing address is wrong; routers can be considered
  as a specialized filter in the Pipes and filters architecture
* Message translator: a conversion of the message, possibly from one format
  to another so that the sender and receiver can use the format that
  are suitable for them
* Message endpoint: the entry/exit point of sender/receiver to the messaging
  system that knows know the application and the messaging system work;
  each endpoint either sends or receives messages, but never both

Each component has its own set of design patterns which Apache Camel
has implemented.

## Interlude: Setting up Apache Camel

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
        implementation("org.apache.camel:camel-$n:3.4.0")
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

* `core-engine`: without which, nothing works; it'll pull in dependencies
  it needs
* `core-languages`: for using scripting languages in building routes with
  Camel's DSL

## Messages
Senders can send messages synchronously (request-response) or
asynchronously (fire-and-forget). Camel differentiates the two in the
message container: [`org.apache.camel.Exchange`][exchange]. `Exchange`
has a `pattern` property that indicates whether the exchange type should
be an `InOut`, `InOnly`, or `InOptionalOut` (enum values from
[`org.apache.camel.ExchangePattern`][exchangePattern]).

### Command Message
Command Message is an implementation of the well-known
[Command pattern][gof-command-pattern] from the Design Patterns book.
It is simply a regular message that happens to contain a command. A
[Simple Object Access Protocol (SOAP)][soap] is a command message.

To ensure each command is only consumed and invoked once, command messages
are usually sent on a [Point-to-Point Channel](#point-to-point-channel).

### Document Message
Document Message is just data the sender sends to the receiver and let
the receiver do whatever it wants to the content. For document messages,
[Guaranteed Delivery](#guaranteed-delivery) is probably more important
than [Message Expiration](#message-expiration).

One of the benefits of sending document messages in a messaging system is that
receivers--when there are more than one expecting the receipt--do not need to
coordinate among themselves to delete the document when they've received
the file; if file transfer were to be used, the receivers need to somehow know
which one of them is the last one picking up the file and be responsible
for deleting it.

Document messages are usually sent using a
[Point-to-Point Channel](#point-to-point-channel), but could be broadcast via
[Publish-Subscribe Channel](#publish-subscribe-channel). Document messages
are usually the replies to the requests in [Request-Reply](#request-reply).

### Event Message
When an application need to notify others of an event, it sends an event
message whose content could be empty or contain anything, e.g., object, XML
document. Unlike [Document Messages](#document-message), Event messages
probably deem [Message Expirations](#message-expiration) to be more important
than [Guaranteed Delivery](#guaranteed-delivery).

Event messages could be delivered via the push or pull model:

* Push: event message contains the information about the change as part of its
  notification.
* Pull: event message has an empty body and requires interested receivers to
  follow up with [Request-Reply](#request-reply) to get the information on the
  change

The disadvantage of push model is that all receivers have to take in
potentially large messages in every update, even if they are not interested
in the content changed. Pull model's disadvantage is that it makes the
applications chatty.

Event messages are usually broadcast via
[Publish-Subscribe Channel](#publish-subscribe-channel). Unlike
[Document Messages](#document-message), subscribers could ignore event messages
if they could not process the event messages in time, therefore, subscribers
are usually not [Durable Subscribers](#durable-subscribers).

### Request-Reply
Because channels are unidirectional, Request-Reply requires a pair of
channels: one for request, the other for reply. The request channel can
be a [Point-to-Point Channel](#point-to-point-channel) or a
[Publish-Subscribe Channel](#publish-subscribe-channel); the reply channel
is almost always point-to-point because it doesn't make sense to broadcast
replies.

Requestor in request-reply could receive the reply in one of the following
two modes:

* Synchronous: the requestor blocks its current thread to wait for reply.
* Asynchronous: the requestor sets up a callback for the reply in a separate
  thread, thus allowing multiple outstanding requests to share a single
  reply channel and reply thread to process replies.

Requests could be a:

* RPC sent as a [Command Message](#command-message) that describes the
  function to invoke and expects a [Document Message](#document-message)
  as reply which could be the return value or the exception thrown
* Query sent as a [Command Message](#command-message) that contains the
  query and expects the reply to be the results of the query as either a
  [Document Message](#document-message), or a
  [Message Sequence](#message-sequence).
* Notify/Acknowledge sent as an [Event Message](#event-message) that notifies
  and expects a [Document Message](#document-message) as acknowledgement
  to the notification.

Request messages should contain a [Return Address](#return-address) to
inform the replier where to send the reply and a
[Correlation Identifier](#correlation-identifier) that specifies which
request this reply is for.

## Message Channels
Most of the time, the number of channels to set up is predefined--agreed
between applications upfront--as opposed to created dynamically and
discovered at runtime. The latter is employed when the sender expects
the receiver to reply to a dynamically created channel by specifying that
in the request message it sends. For all intents and purposes, channels
are unidirectional.

### Point-to-Point Channel
Such channels ensure that only one receiver consumes any given message.
While there could be multiple receivers listening at the end of such
channels, these receivers are known as
[Competing Consumers](#competing-consumers). It is the
job of the point-to-point channel to ensure only one of them consumes the
message. Such design makes the system scalable as it can load-balance across
multiple receivers, without requiring competing consumers to coordinate
among themselves on such process-once-and-only-once behavior.

#### Trying out

Let's use [Apache ActiveMQ 5 "Classic"][activemq] to demonstrate this.
Download ActiveMQ (latest version as of this writing is 5.15.12), unzip it,
and run `bin/activemq start` at the installation directory.[^2] In your browser,
go to `localhost:8161/admin` and sign in with user ID "admin" and password
"admin". Click _Queues_ tab and create a new queue `p2p.sample`.

Add `org.apache.camel:camel-activemq:3.3.0` to the dependency block in
`build.gradle.kts` and run the code below:

```kotlin {linenos="table",hl_lines=[12]}
package eip

import org.apache.camel.builder.RouteBuilder
import org.apache.camel.impl.DefaultCamelContext

fun main() {
    val context = DefaultCamelContext()
    val directURI = "direct:greet"

    context.addRoutes(object: RouteBuilder() {
        override fun configure() {
            val mqURI = "activemq:queue:p2p.sample"

            from(directURI).to(mqURI)
            from(mqURI).to("stream:out")
        }
    })

    context.start()

    val producer = context.createProducerTemplate()

    for (n in 1..10) {
        producer.sendBody(directURI, "Hello, World! for the $n-th time")
    }

    context.stop()
}
```

Line 12 explicitly states the JMS connection is a queue and queues in JMS
are point-to-point channels. By default, `camel-activemq` assumes JMS
destinations are queues, so the string at line 12 could be simplified as
`"activemq:p2p.sample"`. Apache ActiveMQ listens at `localhost:61616` for
incoming messages by default; if it listening from somewhere else, specify
that in the `brokerURL` option, e.g.,
`activemq:p2p.sample?brokerURL=100.200.200.100:8989`

You'll see 10 outputs in standard output after running the above. But if you
comment off line 15 (the routing of ActiveMQ to `stream:out`), run the above,
and examine ActiveMQ console, you'll see 10 pending messages in ActiveMQ, as
shown below:

![Point-to-point demo on ActiveMQ](/images/xxxx-xx-xx-activemq-p2p.png)

### Publish-Subscribe Channel
Publish-Subscribe Channel is (kinda) the opposite of
[Point-to-Point](#point-to-point-channel): it delivers every message in
the channel to all subscribers, ensuring that each subscriber receive
the message once and only once. Publish-Subscribe Channel should only
consider the message is consumed when it has deliver the message to
all the subscribers. Once consumed, Publish-Subscribe Channel can
drop/remove the message from the channel.

#### Trying out

Again, let's use Apache ActiveMQ for this. If you haven't follow the
setup as described in [Point-to-Point Channel](#point-to-point-channel),
but create a new topic `psc.sample` instead. Then run the code below:

```kotlin {linenos="table", hl_lines=[21]}
package eip

import org.apache.camel.builder.RouteBuilder
import org.apache.camel.impl.DefaultCamelContext

class Subscriber(val name: String) {
    fun namedEcho(s: String): String {
        return "$name received: $s"
    }
}


fun main() {
    val context = DefaultCamelContext()
    val foo = Subscriber("foo")
    val bar = Subscriber("bar")
    val directURI = "direct:ps"

    context.addRoutes(object: RouteBuilder() {
        override fun configure() {
            val mqURI = "activemq:topic:psc.sample"

            from(directURI).to(mqURI)
            from(mqURI).bean(foo, "namedEcho").to("stream:out")
            from(mqURI).bean(bar, "namedEcho").to("stream:out")
        }
    })

    context.start()

    val producer = context.createProducerTemplate()
    producer.sendBody(directURI, "Hello, World!")

    context.stop()
}
```

Line 21 specifies the JMS connection is a topic name, i.e., Publish-Subscribe
Channel (otherwise, it's interpreted as a queue). Two subscribers of the
channel prepend the message they receive with `<name> received: ` and outputs
to standard output. `Subscriber` class is a very simple
[Content Enricher](#content-enricher), a kind of
[Message Translator](#message-translator) that we are using for this example
to differentiate the subscribers.

You will see two outputs in standard output, `foo received: Hello, World!`
and `bar received: Hello, World!`. Examining ActiveMQ console, you'll see
one message enqueued and two dequeued:

![Publish-Subscribe demo on ActiveMQ](/images/xxxx-xx-xx-activemq-psc.png)

#### Wildcard Subscribers

While publishers must always publish messages to specific topics,
subscribers may subscribe to multiple channels. Note that different
messaging systems may use different syntax and capabilities for
wildcard subscriptions.

#### Trying out

Again, using ActiveMQ for the following example, create two new topics
`wc.baz` and `wc.quz`. Then run the following code:

```kotlin {linenos="table", hl_lines=[28]}
package eip

import org.apache.camel.builder.RouteBuilder
import org.apache.camel.impl.RouteBuilder

class TopicSubscriber(val name: String) {
    fun namedEcho(s: String): String {
        return "$name received: $s"
    }
}


fun main() {
    val context = DefaultCamelContext()
    val foo = TopicSubscriber("foo")
    val bar = TopicSubscriber("bar")
    var directBazURI = "activemq:topic:wc.baz"
    var directQuzURI = "activemq:topic:wc.quz"

    context.addRoutes(object: RouteBuilder() {
        override fun configure() {
            val mqBazURI = "activemq:topic:wc.baz"
            val mqQuzURI = "activemeq:topic:wc.quz"

            from(directBazURI).to(mqBazURI)
            from(directQuzURI).to(mqQuzURI)

            from(mqBazURI).bean(foo, "namedEcho").to("stream:out")
            from("activemq:topic:wc.*").bean(bar, "namedEcho").to("stream:out")
      }
    })

    context.start()

    val producer = context.createProducerTemplate()
    producer.sendBody(directBazURI, "Hello, World!")
    producer.sendBody(directQuzURI, "Goodbye, Universe!")

    Thread.sleep(100)
    context.stop()
}
```

Notice that the topic name that routes the message to `bar` bean uses
the wildcard `*` in its destination. This means `bar` bean has subscribed
to both `wc.baz` and `wc.quz`. `foo` bean has subscribed only to
`wc.baz` because it specifies the exact destination it's interested in.
You'll see the following in standard output:

```bash
foo received: Hello, World!
bar received: Hello, World!
bar received: Goodbye, Universe!
```

### Datatype Channel

Sometimes, it's useful to set up multiple channels for the sender to
send messages of different datatype and format to different channels.
This is analogous to having a collection with homogeneous objects; such
strategy makes its easier for receivers to know the datatype of the message
it reads off the channel.

However, putting different messages with different datatype on the channel
because it is not economical to create too many channels when a sender needs
to send a wide variety of messages of different datatype. There are multiple
ways to help the receivers at the end of such channels know how to consume
messages:

1. Receiver uses [Selective Consumer](#selective-consumer) to process
   only messages they want.
2. Message system adds a [Content-Based Router](#content-based-router)
   to route the messages by datatype to receivers.
3. Sender employs [Format Indicator](#format-indicator) in the message's
   header to specify the message format.
4. Sender wraps the data in a [Command Message](#command-message) with
   different command for each type of data and presumes receivers know
   what to do with the data.

Datatype Channel explains why messages in the channel must have the same data
format, which is different from [Canonical Data Model](#canonical-data-model)
that explains how all messages on all channels should follow the unified
data model.

### Invalid Message Channel

When a receiver receive a message that it doesn't know how to process,
it should reroute that message to an Invalid Message Channel for diagnostics.
The receiver's context determines the message validity: a message may be
valid to one receiver but invalid to another, e.g., the first receiver
expects the message to be in CSV format, but the second expects it to be in
XML. In such cases, it could be a wrongly configured sender putting the
wrong message to the wrong channel. When a receiver receives a message
that should contain certain data but doesn't, the receiver should also
route such messages to the Invalid Message Channel, e.g., a message
that should have an mailing address field but doesn't.

Note that when one of the multiple receivers at the end of a channel
determines that the message is invalid, we should consider that message
to be invalid for all other receivers at the end of the same channel.
When a message is valid to one receiver but not to the other, these two
receivers should be on different channels.

Be careful not to treat valid messages with semantically incorrect contents
as invalid messages. For example, a sender may put a
[Command Message](#command-message) to instruct the receiver to delete
a record that does not exist. Such is not a messaging error but an
application error, thus the receiver should not route such messages to
the Invalid Message Channel but handle it as an invalid request (not
invalid message).

### Dead Letter Channel

A close cousin to [Invalid Message Channel](#invalid-message-channel), Dead
Letter Channel is where the messaging system routes messages that it couldn't
deliver to. In the case of [Invalid Message Channel](#invalid-message-channel),
the messaging system could deliver the message but the receiver could not
process it due to invalidity. Most, if not all, messaging systems implements
their version of Dead Letter Channel. Camel implements this pattern by
swapping the default error handler with `DeadLetterChannel`.

#### Trying out

Create the queue `dead` in ActiveMQ, then run the code below:

```kotlin {linenos="table", hl_lines=[14,15,16,18]}
package eip

import org.apache.camel.builder.RouteBuilder
import org.apache.camel.impl.DefaultCamelContext

class NoOp

fun main() {
    val context = DefaultCamelContext()
    val directURI = "direct:dlc"

    context.addRoutes(object: RouteBuilder() {
        override fun configure() {
            val dlq = deadLetterQueue("activemq:queue:dead")
                .maximumRedeliveries(3)
                .redeliveryDelay(500)

            errorHandler(dlq)
            from(directURI).bean(NoOp::class.java).to("stream:out")
        }
    })

    context.start()
    val produceer = context.createProducerTemplate()
    producer.sendBody(directURI, "Dead on arrival")

    Thread.sleep(100)
    context.stop()
}
```

Lines 14-16 configures the `DeadLetterChannel` to reroute dead messages
to `activemq:queue:dead` if the messaging system still fails to deliver
the message after 3 retries (each retry has 500 milliseconds delay
in-between). This then is used as the error handler (line 18).[^3]
You shouldn't see the output `Dead on arrival` in the standard output.
Camel has routed the message to the dead letter queue that you
can examine in ActiveMQ's console: click the queue name, you'll see one
message stuck in the queue.

![ActiveMQ Dead Letter Queue Summary](/images/xxxx-xx-xx-activemq-dlc.png)

When you click the message, you'll see the message `Dead on arrival`
in the _Message Details_ section at the bottom:

![ActiveMQ Dead Letter Queue Message Content](/images/xxxx-xx-xx-activemq-dlc-message.png)

### Guaranteed Delivery

Messaging systems and receivers may fail too, i.e., they may not be online
all the time. Therefore, Guaranteed Delivery makes messages persistent so
that they aren't lost even when the messaging system or receiver are not
available. When receivers are offline, the messaging system will likely
fail in redelivery, so persisting the messages may be necessary to prevent
the case of the messaging system crashes too.

By default, JMS-compliant systems persist messages; while this default is
likely necessary in production, it's much easier to test and debug with
persistence off to prevent the test/debug session from processing
messages persisted at the end of the previous session. The ActiveMQ screenshot
at the end of [Point-to-Point Channel](#point-to-point-channel) shows 10
messages are pending processing. If that were to be running in production,
then good, nothing is lost; but if that were to be a testing/debugging
session, the session will be processing messages from the previous one. One
wonders where those extra messages come from!

### Channel Adapter

Channel Adapter allows applications that aren't designed for integration
with messaging systems to be able to connect to one. Often, it may be
impossible to customize these applications due to the lack of access
to the source code or due to the lack of deep understanding of the
application logic and messaging API. Note that simply making the application
output a file for putting into the messaging system does not make that
a Channel Adapter.

There are three ways to realize a Channel Adapter:

1. User Interface Adapter: aka screen scraping, uses the application's
   screen--like a human would--and passes information it scrapes from the
   screen to the messaging system; such adapter may be slow and brittle as
   it needs to parse and scrape from UIs which usually have more
   frequent changes
2. Business Logic Adapter: wraps components the application exposes in
   the form of COM objects, CORBA components, EJBs, libraries, HTTP APIs, etc;
   usually, such adapting is the best choice because the vendor (internal
   or otherwise) are expressly exposing the APIs for others to use
3. Database Adapter: extracts data directly from the application's database,
   with possibly real-time interfacing to the messaging system by adding
   triggers in the database for notifications of record updates; while this
   is the least invasive, this require intimate knowledge of the application's
   database schema which vendors normally consider these as unpublished and
   reserve the rights to change it at will.

Apache Camel supports this pattern through its plethora of
[components][components], e.g., it has a [Twilio component][camel-twilio]
since version 2.20 that adapts Twilio REST APIs for Camel use.

Channel adapters suffer from the need to convert the messages to resemble
the implementation of the component being adapted. For example, a database
adapter requires incoming messages to use the same names and fields of
the tables. Such coupling hurts integration, so channel adapters are usually
paired with [Message Translator](#message-translator) to convert
application-specific messages into messages that complies with the
[Canonical Data Model](#canonical-data-model).

There are two special forms of channel adapters:

1. Metadata adapter: such adapters expose metadata of the application they
   adapt, e.g., system tables in databases that describe the application tables.
2. [Messaging Bridge](#messaging-bridge): these connect one messaging system
   to another messaging system.

#### Trying out

Let's use SQLite to demonstrate using a [JDBC][camel-jdbc] channel adapter. Add
`org.xerial:sqlite-jdbc:3.32.3.1` to the dependency block in `build.gradle.kts`
and run the code below:

```kotlin {linenos=table, hl_lines=[13,25]}
package eip

import javax.sql.DataSource
import org.apache.camel.builder.RouteBuilder
import org.apache.camel.impl.DefaultCamelContext
import org.sqlite.SQLiteDataSource


fun main() {
    val datasource = setupDatabase()
    val context = DefaultCamelContext()

    context.registry.bind("myDS", datasource)

    context.addRoutes(object: RouteBuilder() {
        override fun configure() {
            from("direct:trigger").to("jdbc:myDS")
        }
    })

    context.start()

    val producer = context.createProducerTemplate()

    producer.sendBody("direct:trigger", "INSERT INTO foo VALUES('hi', 'world')")
    readOneRowFromFoo(datasource)

    context.stop()
}


private fun setupDatabase(): DataSource {
    val ds = SQLiteDataSource()
    ds.setUrl("jdbc:sqlite:./build/channel-adapter.db")

    ds.getConnection().use { conn ->
        val stmt = conn.createStatement()
        stmt.execute("DROP TABLE IF EXISTS foo")
        stmt.execute("CREATE TABLE foo (bar, baz)")
    }

    return ds
}


private fun readOneRowFromFoo(datasource: DataSource) {
    datasource.getConnection().use { conn ->
        val stmt = conn.createStatement()
        val resultSet = stmt.executeQuery("SELECT bar, baz FROM foo")
        resultSet.next()

        println("bar=${resultSet.getString(1)}, baz=${resultSet.getString(2)}")
    }
}
```

This is a big chunk of code, so let's break it down a little to understand.
After calling the private function `setupDatabase` to create the table `foo`,
line 13 registers the returned data source in Apache Camel's registry as
`myDS`.[^4] Registration--however it is done--is necessary, as it allows
Apache Camel to look up resources the components need.
Line 25 send an SQL statement as the body to the JDBC channel
adapter via `direct:trigger` to insert a row into the table `foo`.
Immediately after that, it reads one row from the table and outputs the
result. You should see `bar=hi, baz=world` in standard output.

### Messaging Bridge

A specialized form of [Channel Adapter](#channel-adapter), messaging bridge
connects one messaging system to another. Typically, they also translate
message formats from one messaging solution to the other, e.g., translate
Java MQ message format in source messaging system to MSMQ message format
in receiving messaging system. Apache Camel achieves this in the route builder:

```kotlin {linenos=table}
    from("mq:queue:foo").to("activemq:queue:foo")
```

### Message Bus

Message bus is an integration architecture that enables applications to
coordinate in loosely coupled way via messaging. Message bus allows
applications to continue running with minimal impact when some others are not.
This also allows applications to be added to or removed from the enterprise
with minimal impact.

Message bus is a combination of a common data model, a common command set, and
a messaging infrastructure to allow different systems to communicate through
a shared set of interfaces. A message bus forms a simple, yet useful
service-oriented architecture, where each service has at least one request
channel that accepts requests of an agreed format, and probably a
corresponding reply channel that supports a specified reply format.

All applications using the message bus must use the same
[Canonical Data Model](#canonical-data-model); they may also need to depend
on [Message Routers](#message-router) to route the message to the appropriate
destination. Applications not designed to interface with a messaging system
may require [Channel Adapters](#channel-adapter) and
[Service Activators](#service-activator).

[^1]: [Camel in Action, 2nd Edition][cia2e] actually did recommend
      reading the EIP book shortly into chapter 1, but I ignored that
      advice :sweat_smile:
[^2]: To stop, run `bin/activemq stop` in the installation directory.
[^3]: If there's not error handler specified, Camel uses the
      [DefaultErrorHandler][defaulterrorhandler].
[^4]: In our case, we're using `org.apache.camel.support.DefaultRegistry`.

[camel]: https://camel.apache.org
[eip]: https://www.enterpriseintegrationpatterns.com
[cia2e]: https://www.manning.com/books/camel-in-action-second-edition
[exchange]: https://www.javadoc.io/doc/org.apache.camel/camel-api/3.3.0/org/apache/camel/Exchange.html
[exchangePattern]: https://www.javadoc.io/doc/org.apache.camel/camel-api/3.3.0/org/apache/camel/ExchangePattern.html
[gof-command-pattern]: https://en.wikipedia.org/wiki/Command_pattern
[soap]: https://en.wikipedia.org/wiki/SOAP
[activemq]: https://activemq.apache.org
[defaulterrorhandler]: https://camel.apache.org/manual/latest/error-handler.html#_defaulterrorhandler
[components]: https://camel.apache.org/components/latest/index.html
[camel-twilio]: https://camel.apache.org/components/latest/twilio-component.html
[camel-jdbc]: https://camel.apache.org/components/latest/jdbc-component.html
