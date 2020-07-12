---
title: "Getting Started With Chronicle Queue"
date: 2020-04-26T16:30:00+08:00
allowComments: true
tags: [kotlin, chronicle queue]
---

[Chronicle Queue][cq] is low-latency, broker-less, durable message queue.
Its closest cousin is probably [0MQ][zmq], except that 0MQ doesn't store
the messages published and the open-source version of Chronicle Queue doesn't
support cross-machine communication[^1]. Chronicle Queue's biggest claim to fame
is that it generates no garbage as it uses `RandomAccessFile`s as off-heap
storage.

Chronicle Queue is producer-centric, i.e., applications built on Chronicle
Queue cannot tell the producer to slow down in putting messages onto
the queue (no back-pressure mechanics). Such design is useful in cases
where there is little to no control on the producer's throughput, e.g.,
FX price updates.

## Terminology

Where most message queues use the terms Producer and Consumer, Chronicle
Queue uses Appender and Tailer instead to make the distinction that it
always appends messages to the queue and it never "destroys/drops" any
message after the tailer (read: receiver) reads the message from the queue.
And instead of Message, Chronicle Queue prefers the term Excerpt because
the blob written to Chronicle Queue can range from byte arrays to strings
to domain models.

## Hello, World!

Let's use the traditional "Hello, World!" to demonstrate basic usage. Add
the following to `build.gradle.kts` if you are using Gradle:

{{< highlight kotlin "linenos=table, hl_lines=1 17 18 25-27">}}
import org.jetbrains.kotlin.gradle.tasks.KotlinCompile

plugins {
    id("org.jetbrains.kotlin.jvm") version "1.3.71"
    application
}

repositories {
    mavenCentral()
    mavenLocal()
}

dependencies {
    implementation("org.jetbrains.kotlin:kotlin-bom")
    implementation("org.jetbrains.kotlin:kotlin-stdlib-jdk8")

    implementation("net.openhft.chronicle:chronicle-queue:5.19.8")
    implementation("org.apache.logging.log4j:log4j-slf4j18-impl:2.13.1")
}

application {
    mainClass = "hello.AppKt"
}

tasks.withType<KotlinCompile> {
    kotlinOptions.jvmTarget = "1.8"
}
{{</ highlight >}}

Importing `KotlinCompile` (line 1) allows specifying Java 1.8 as the
compilation target (lines 25-27)[^2]. Lines 17-18 shows the additional
dependencies you'd need to get started with Chronicle Queue. Note that
`build.gradle.kts` assumes the package to use is `hello`. Let's turn to
the code demonstrating Chronicle Queue usage:

{{< highlight kotlin "linenos=table">}}
package hello

import net.openhft.chronicle.queue.ChronicleQueue

fun main(args: Array<String>) {
    val q: ChronicleQueue = ChronicleQueue.single("./build/hello-world")

    try {
        val appender: ExcerptAppender = q.acquireAppender()
        appender.writeText("Hello, World!")

        val tailer: ExcerptTailer = q.createTailer()
        println(tailer.readText())
    } finally {
      q.close()
    }
}
{{</ highlight >}}

`ChronicleQueue.single(<path>)` returns a new `ChronicleQueue`
that uses the given path for storing the excerpts. The rest of the code
are pretty much self-explanatory: the acquired appender appends
the excerpt `"Hello, World!"` to the queue; the tailer reads from the
queue and prints the excerpt to standard output. The queue must always be
closed at the end of the program.[^3]

Remember that Chronicle Queues are durable? Comment out two appender lines
and run the code again with `gradle run`. You'll see that the program
outputs `Hello, World!` again in standard output: the tailer is reading
from the queue that was written in the previous run. Such durability
allows replaying incoming excerpts when tailers crash.

## Detour: Excerpt types

Chronicle Queue only accepts the following types as excerpts:

1. `Serializable` objects: note that serializing such objects are
   inefficient due to reliance on reflection
2. `Externalizable` objects: if compatibility with Java is important but
   at the expense of handwritten logic
3. `byte[]`
4. `String`
5. `net.openhft.chronicle.wire.Marshallable` objects: high performance data
   exchange using binary formats
6. `net.openhft.chronicle.bytes.BytesMarshallable` objects: low-level binary
   or text encoding

As "Hello, World!" has already demonstrated strings, we detour a little
and look at an example using `Marshallable` offered in [Chronicle Wire][cw]
library.

{{< highlight kotlin "linenos=table">}}
package types

import net.openhft.chronicle.wire.Marshallable
import net.openhft.chronicle.wire.SelfDescribingMarshallable

class Person(val name: String, val age: Int): SelfDescribingMarshallable()

fun main(args: Array<String>) {
    val person = Person("Shaolang", 3)
    val outputString = """
    !types.Person {
      name: Shaolang
      age: 3
    }

    """.trimIndent()

    println(person.toString() == outputString)

    val p = Marshallable.fromString<Person>(outputString)

    println(person == p)
    println(person.hashCode() == p.hashCode())
}
{{</ highlight >}}

You'll see three `true` printed to standard output when you run the
snippet above. `SelfDescribingMarshallable` makes it effortless to
make a class `Marshallable` for persistence in Chronicle Queue.

## Writing and Reading Domain Objects

With the knowledge from the small detour tucked under our belt, the
following demonstrates the writing and reading of `Marshallable`
objects to and from Chronicle Queue[^4]:

{{< highlight kotlin "linenos=table">}}
package docs

import net.openhft.chronicle.queue.ChronicleQueue
import net.openhft.chronicle.wire.SelfDescribingMarshallable

class Person(var name: String? = null, var age: Int? = null): SelfDescribingMarshallable()

class Food(var name: String? = null): SelfDescribingMarshallable()

fun main(args: Array<String>) {
    ChronicleQueue.single("./build/documents").use { q ->
        val appender = q.acquireAppender()

        appender.writeDocument(Person("Shaolang", 3))
        appender.writeText("Hello, World!")
        appender.writeDocument(Food("Burger"))

        val tailer = q.createTailer()

        val person = Person()
        tailer.readDocument(person)
        println(person)

        println("${tailer.readText()}\n")

        val food = Food()
        tailer.readDocument(food)
        println(food)
    }
}
{{</ highlight >}}

After running the above, you should see the following printed out:

```bash
!docs.Person {
  name: Shaolang,
  age: 3
}

Hello, World!

!docs.Food {
  name: Burger,
}
```

There's a few things to note:

1. Because Chronicle Queue aims to generate no garbage, it requires
   the domain model to be a mutable object; this is why the two classes
   uses `var` instead of `val` in their constructors.
2. Chronicle Queue allows appenders to write different things to the
   same queue.
3. Tailers need to know what it should be reading to get the proper
   result back.

If we were to change the last `tailer.readDocument(food)` to
`tailer.readDocument(person)` and print out `person` instead, we'll see
the following printed[^5]:

```bash
!docs.Person {
  name: Burger,
  age: !!null ""
}
```

Because both `Person` and `Food` have an attribute with the same name,
Chronicle Queue hydrates `Person` with whatever it could and leave the
others blank.

That last point on tailers needing to know what they're reading is
troubling: they are now ladened with the burden of filtering things
they want to be notified from the avalanche of data the producer keep
throwing at them. To keep our codebases sane, we need to use the
[observer pattern][observer].

## (Kinda) Listening Only to Things You're Interested in

Other than using the excerpt appender directly, another way is to make it
reify the first class given to its `methodWriter`. The following snippet
focuses on this reifying of the given listener:

{{< highlight kotlin "linenos=table, hl_lines=17-18" >}}
package listener

import net.openhft.chronicle.queue.ChronicleQueue
import net.openhft.chronicle.queue.ChronicleReaderMain
import net.openhft.chronicle.wire.SelfDescribingMarshallable

class Person(var name: String? = null, var age: Int? = null): SelfDescribingMarshallable()

interface PersonListener {
    fun onPerson(person: Person)
}

fun main(args: Array<String>) {
    val directory = "./build/listener"

    ChronicleQueue.single(directory).use { q ->
        val observable: PersonListener = q.acquireAppender()
              .methodWriter(PersonListener::class.java)
        observable.onPerson(Person("Shaolang", 3))
        observable.onPerson(Person("Elliot", 4))
    }

    ChronicleReaderMain.main(arrayOf("-d", directory))
}
{{</ highlight >}}

Lines 17-18 invoke `methodWriter` with the given `PersonListener` on the
acquired appender. Notice that the type assigned to `observable` is
`PersonListener`, not `ExcerptAppender`. Now, any calls to the
methods in `PersonListener` writes the given argument to the queue.
However, there's a difference in writing to the queue using the appender
directly and using a reified class. To see the difference, we'll use
`ChronicleReaderMain` to examine the queue:

```bash
0x47c900000000:
onPerson {
  name: Shaolang,
  age: 3
}

0x47c900000001:
onPerson {
  name: Elliot,
  age: 4
}
```

Notice that instead of `!listener.Person { ... }`, reified classes
write excerpts using `onPerson {...}` to the queue. This difference
allows tailers that implement `PersonListener` to be notified of
new `Person` objects written to the queue and ignore others that they
aren't interested in.

Yup, you've read that right: tailers that implement `PersonListener`.
Unfortunately, Chronicle Queue (kinda) conflates observables and observers,
thus making it a little hard in distinguishing observables from
observers. I think the easiest way to tell the difference is to use
the heuristics as shown in the following snippet's comments:

{{< highlight kotlin >}}
interface PersonListener {
    onPerson(person: Person)
}

// this is an observer because it implements the listener interface
class PersonRegistry: PersonListener {
    override fun onPerson(person: Person) {
        // code omitted for brevity
    }
}

fun main(args: Array<String>) {
    // code omitted for brevity

    val observable: PersonListener = q.acquireAppender()    // this is an
            .methodWriter(PersonListener::class.java)       // observable

    // another way to differentiate: the observer will never call the
    // listener method, only observables do.
    observable.onPerson(Person("Shaolang", 3))

    // code omitted for brevity
}
{{</ highlight >}}

Let's turn our focus to tailers. Even though Chronicle Queue ensures that
[every tailer sees every excerpt][etsem], tailers can filter only
excerpts that they want to see by implementing the listener class/interface
and creating a `net.openhft.chronicle.bytes.MethodReader` with the
implemented listener:

{{< highlight kotlin "linenos=table, hl_lines=31" >}}
package listener

import net.openhft.chronicle.bytes.MethodReader
import net.openhft.chronicle.queue.ChronicleQueue
import net.openhft.chronicle.wire.SelfDescribingMarshallable

class Person(var name: String? = null, var age: Int? = null): SelfDescribingMarshallable()

class Food(var name: String? = null): SelfDescribingMarshallable()

interface PersonListener {
    fun onPerson(person: Person)
}

class PersonRegistry: PersonListener {
    override fun onPerson(person: Person) {
        println("in registry: ${person.name}")
    }
}

fun main(args: Array<String>) {
    ChronicleQueue.single("./build/listener2").use { q ->
        val appender = q.acquireAppender()
        val writer: PersonListener = appender.methodWriter(PersonListener::class.java)

        writer.onPerson(Person("Shaolang", 3))
        appender.writeDocument(Food("Burger"))
        writer.onPerson(Person("Elliot", 4))

        val registry: PersonRegistry = PersonRegistry()
        val reader: MethodReader = q.createTailer().methodReader(registry)

        reader.readOne()
        reader.readOne()
        reader.readOne()
    }
}
{{</ highlight >}}

What's largely new to this is the implementation of `PersonRegistry` that
simply prints the name of the person it is given. Instead of reading off
the queue using an `ExcerptTailer` directly, the snippet creates a
`MethodReader` from the tailer with the given instantiated `PersonRegistry`.
Unlike `.methodWriter` that accepts `Class`, `.methodReader` expects objects.
The appender writes three excerpts to the queue: person (via call
to `onPerson`), food (via `.writeDocument`), and person.
Because tailers see every excerpts, the reader also make three calls
to "read" all excerpts, but you'll see only two outputs:

```bash
in registry: Shaolang
in registry: Elliot
```

If only there were only two `.readOne()` calls instead of three, the output
will not include `in registry: Elliot`.

### `MethodReader` Uses Duck Typing

Remember the outputs from `ChronicleReaderMain` when we examined the
queue that's populated by the reified `PersonListener`? Instead of
a class name, the outputs are similar to `onPerson { ... }`. That
suggests `MethodReader` filters excerpts that matches the method signature,
i.e., it doesn't care about the interface/class that contains
the method signature; or simply put, [duck typing][duck-typing]:

{{< highlight kotlin "linenos=table,hl_lines=16" >}}
package duck

import net.openhft.chronicle.queue.ChronicleQueue
import net.openhft.chronicle.wire.SelfDescribingMarshallable

class Person(var name: String? = null, var age: Int? = null): SelfDescribingMarshallabl()

interface PersonListener {
    fun onPerson(person: Person)
}

interface VIPListener {
    fun onPerson(person: Person)
}

class VIPClub: VIPListener {
    override fun onPerson(person: Person) {
        println("Welcome to the club, ${person.name}!")
    }
}

fun main(args: Array<String>) {
    ChronicleQueue.single("./build/duck").use { q ->
        val writer = q.acquireAppender().methodWriter(PersonListener::class.java)
        writer.onPerson(Person("Shaolang", 3))

        val club = VIPClub()
        val reader = q.createTailer().methodReader(club)

        reader.readOne()
    }
}
{{</ highlight >}}

Notice that `VIPClub` implements `VIPListener` that happens to have the
same `onPerson` method signature as `PersonListener`. When you run the
above, you'll see `Welcome to the club, Shaolang!` printed.

## Named Tailers

In all demonstrations so far, we've been creating anonymous tailers.
Because they are anonymous, every (re-)run results in reading all
excerpts in the queue. Sometimes, such behavior is acceptable, or
even desirable, but there are times it doesn't. To pick up reading
where it last stopped is done simply by naming the tailer:

{{< highlight kotlin "linenos=table, hl_lines=8" >}}
package restartable

import net.openhft.chronicle.queue.ChronicleQueue
import net.openhft.chronicle.queue.ExcerptTailer

fun readQueue(tailerName: String, times: Int) {
    ChronicleQueue.single("./build/restartable").use { q ->
        val tailer = q.createTailer(tailerName)       // tailer name given

        for (_n in 1..times) {
          println("$tailerName: ${tailer.readText()}")
        }

        println()       // to separate outputs for easier visualization
    }
}

fun main(args: Array<String>) {
    ChronicleQueue.single("./build/restartable").use { q ->
        val appender = q.acquireAppender()
        appender.writeText("Test Message 1")
        appender.writeText("Test Message 2")
        appender.writeText("Test Message 3")
        appender.writeText("Test Message 4")
    }

    readQueue("foo", 1)
    readQueue("bar", 2)
    readQueue("foo", 3)
    readQueue("bar", 1)
}
{{</ highlight >}}

Notice that the tailer's name is given to `createTailer` method. The code
above has two tailers--unimaginatively named `foo` and `bar`--reading
off the queue and outputs the following when run:

```bash
foo: Test Message 1

bar: Test Message 1
bar: Test Message 2

foo: Test Message 2
foo: Test Message 3
foo: Test Message 4

bar: Test Message 3
```

Notice that the second time foo and bar read from the queue, they
pick up from where they've left previously.

## Roll 'Em

Chronicle Queue rolls the file it uses based on the roll cycle defined
when the queue is created; by default, it rolls the file daily. To
change the roll cycle, we cannot use the simple `ChronicleQueue.single`
method anymore:

{{< highlight kotlin "linenos=table, hl_lines=8 10-11" >}}
package roll

import net.openhft.chronicle.queue.ChronicleQueue
import net.openhft.chronicle.queue.RollCycles
import net.openhft.chronicle.impl.single.SingleChronicleQueueBuilder

fun main(args: Array<String>) {
    var qbuilder: SingleChronicleQueueBuilder = ChronicleQueue.singleBuilder("./build/roll")

    qbuilder.rollCycle(RollCycles.HOURLY)
    val q: ChronicleQueue = qbuilder.build()

    // code omitted for brevity
}
{{</ highlight >}}

First, we get an instance of `SingleChronicleQueueBuilder` and set the roll
cycle with `.rollCycle` method. The snippet above configures the queue to
roll the file hourly. When we are happy with the configuration, call `.build()`
on the builder to get an instantiated `ChronicleQueue`. Note that both
appender and tailer(s) must use the same roll cycle when accessing the same
queue.

As `SingleChronicleQueueBuilder` supports fluent interface, the code could
also be simplified as follows:

```kotlin
val q: ChronicleQueue = ChronicleQueue.singleBuilder("./build/roll")
                                      .rollCycle(RollCycles.HOURLY)
                                      .build()
```

## What's Next

This post covers Chronicle Queue terminology and basics. The following
sites have more information to dig from:

1. [Chronicle Queue GitHub repository][cq]
2. [Stack Overflow tagged questions][so]
3. [Peter Lawrey's Blog][plawrey]

Have fun!

[^1]: I wonder whether it's possible and make sense to combine the two to
get the best of both worlds.
[^2]: As of this writing, Kotlin JVM compiler produces Java 6 compatible
bytecode, as stated in
https://kotlinlang.org/docs/reference/faq.html#what-does-kotlin-compile-down-to
[^3]: The code could be simplified further by replacing try-finally construct
with Kotlin's equivalent of Java's try-with-resource syntax.
[^4]: Although it'll make more sense to run the appender and tailer in
different VM processes, keeping both in the same VM makes it much simpler
to understand the discussion without having to sieve through non-related
code.
[^5]: At least as of Chronicle Queue version 5.19.x, it doesn't crash/throw
any exceptions.

[cq]: https://github.com/OpenHFT/Chronicle-Queue
[zmq]: https://zeromq.org
[cw]: https://github.com/OpenHFT/Chronicle-Wire
[observer]: https://en.wikipedia.org/wiki/Observer_pattern
[etsem]: https://github.com/OpenHFT/Chronicle-Queue#every-tailer-sees-every-message
[duck-typing]: https://en.wikipedia.org/wiki/Duck_typing
[so]: https://stackoverflow.com/questions/tagged/chronicle-queue
[plawrey]: https://vanilla-java.github.io
