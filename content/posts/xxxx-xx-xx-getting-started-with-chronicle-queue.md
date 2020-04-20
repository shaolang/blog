---
title: "Getting Started With Chronicle Queue"
date: 2020-04-20T18:44:37+08:00
draft: true
---

[Chronicle Queue][cq] is low-latency, broker-less, durable message queue.
Its closest cousin is probably [0MQ][zmq], except that 0MQ doesn't store
the messages published and the open-source version of Chronicle Queue doesn't
support cross-machine communication. Chronicle Queue's biggest claim to fame
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

{{< highlight kotlin "linenos=table, hl_lines=1 16 17 24-26">}}
import org.jetbrains.kotlin.gradle.tasks.KotlinCompile

plugins {
    id("org.jetbrains.kotlin.jvm") version "1.3.71"
    application
}

repositories {
    mavenCentral()
}

dependencies {
    implementation("org.jetbrains.kotlin:kotlin-bom")
    implementation("org.jetbrains.kotlin:kotlin-stdlib-jdk8")

    implementation("net.openhft.chronicle:chronicle-queue:5.19.8")
    implementation("org.apache.logging.log4j:log4j-sl4fj18-impl:2.13.1")
}

application {
    mainClass = "hello.AppKt"
}

tasks.withType<KotlinCompile> {
    kotlinOptions.jvmTarget = "1.8"
}
{{</ highlight >}}

Importing `KotlinCompile` (line 1) allows specifying Java 1.8 as the
compilation target (lines 24-26). Lines 16-17 shows the additional
dependencies you'd need to get started with Chronicle Queue. Note that
`build.gradle.kts` assumes the package to use is `hello`. Let's turn to
the code demonstrating Chronicle Queue usage:

{{< highlight kotlin "linenos=table">}}
package hello

import net.openhft.chronicle.queue.ChronicleQueue

fun main(args: Array<String>) {
    val q = ChronicleQueue.singleBuilder("./build/hello-world").build()

    try {
        val appender = q.acquireAppender()
        appender.writeText("Hello, World!")

        val tailer = q.createTailer()
        println(tailer.readText())
    } finally {
      q.close()
    }
}
{{</ highlight >}}

`ChronicleQueue.singleBuilder(<path>).build()` returns a new Chronicle Queue
that uses the given path for storing the excerpts. To customize the
queue, `.singleBuilder` returns an instance of `SingleChronicleQueueBuilder`
that exposes many configuration method. For this demonstration, we keep
it as simple as possible.

The rest are pretty much self-explanatory: the acquired appender appends
the excerpt `"Hello, World!"` to the queue; the tailer reads from the
queue and prints the excerpt to standard output. The queue is always closed
at the end of the program.[^1]

Remember that Chronicle Queues are durable? Comment out two appender lines
and run the code again with `gradle run`. You'll see that the program
outputs `Hello, World!` again in standard output: the tailer is reading
from the queue that was written in the previous run. Such durability
allows replaying incoming excerpts when tailers crash.

## Detour: Excerpt types

Chronicle Queue only accepts the following types as excerpts:

1. `Serializable` objects: note that such objects are inefficient
2. `Externalizable` objects: if compatibility with Java is important
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


[^1]: The code could be simplified further by replacing try-finally construct
with Kotlin's equivalent of Java's try-with-resource syntax.

[cq]: https://github.com/OpenHFT/Chronicle-Queue
[zmq]: https://zeromq.org
[cw]: https://github.com/OpenHFT/Chronicle-Wire
