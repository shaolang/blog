---
title: "Getting Started With Apache Camel"
date: 2020-03-23T13:06:37+08:00
draft: true
---

[Apache Camel][camel] is an excellent [enterprise integration][ei]
framework. The canonical reference [Camel in Action, 2nd Edition][cia],
though excellent, shows code that's not runnable (unless I've set up
all the necessary stuff, such as a JMS queue). And it's also based
on version 2.x, so as a beginner, I'm documenting my learning journey,
but with version 3.1.x.

## Hello, World!

I'm using Kotlin and Gradle in the examples below, but translating them
to Java/Scala and Maven/Ant should be relatively simple. Let's start with
some code first:

{{< highlight kotlin "linenos=table, hl_lines=14" >}}
// in file t.App.kt

package t

import org.apache.camel.CamelContext
import org.apache.camel.ProducerTemplate
import org.apache.camel.builder.RouteBuilder
import org.apache.camel.impl.DefaultCamelContext

fun main(args: Array<String>) { // public static void main(String... args) in Java-speak
    val context: CamcelContext = DefaultCamelContext()  // instantiate new camel context
    context.addRoutes(object : RouteBuilder() {         // similar to instantiating anonymous class
        override fun configure() {
            from("direct:greet").to("stream:out")       // routes direct producer to stdout
        }
    })

    context.start()                                     // starts the context
    val template: ProducerTemplate = context.createProducerTemplate()
    template.sendBody("direct:greet", "Shaolang")       // interact with context by sending message to it directly
    context.stop()
}
{{</ highlight >}}


That short snippet is a lot more verbose than necessary, e.g., Kotlin can
infer omitted type in variable declarations; spelling out the types makes it
clear the interface/class most code should depend on, as opposed to the
concrete type. In any case, the snippet still
shows a lot of things: highlighted line 14 connects
[Direct Component][direct-comp] to `stdout` (via
[Steam Component][stream-comp]). The first component allows direct invocation
to an endpoint when a producer sends a message (line 20). The second
component grants access to `System.in`, `System.out`, and `System.err`.

To run this example using Gradle, add the following dependencies:

```kotlin
dependencies {
    // other dependencies are omitted for brevity

    for (s in listOf("core-engine", "direct", "stream")) {
      implementation("org.apache.camel:camel-${s}:3.1.0")
    }

    implementation("org.slf4j:slf4j-nop:1.7.30")  // so logs won't obfuscate stdout
}
```

`camel-core-engine` is the minimal required to run Camel; `camel-direct` and
`camel-stream` add the two endpoint components required by the code's route.
Running `gradle run` should show the output "Shaolang".

Both `direct:greet` and `stream:out` are endpoints, i.e., the former
is the entry point and the latter the exit point. Endpoints are encoded
in URIs in the form of `<scheme>:<context path>?<options>`. Both of our
endpoints don't specify any options. The schema and context path for
`direct:greet` are `direct` and `greet` respectively; Direct Component's
documentation states that all such components uses `direct` as its schema.
However, its context path can be any string without blank spaces in-between.

Unlike `direct:greeet`, `stream:out` is stricter: Stream Component's schema
is `stream` and it accepts only following context paths:

* `in`: accepts input from standard input
* `out`: prints to standard output
* `err`: prints to standard error
* `header`: uses the header field in the message to determine the stream to
  write to (i.e., this context cannot appear in `from()`)
* `file`: prints to the file stated in the given option

That's great, but all the trouble just to do that?

## Adding processors to the routes

Processors can be added to transform messages. For example, to
append (transform) the phrase "Hello" to the input, we could add a processor
in JavaBean convention between the endpoints:

{{< highlight kotlin "linenos=table, linenostart=24" >}}
class HelloBean {
    fun sayGreeting(name: String): String {
        return "Hello, $name!"
    }
}
{{</ highlight >}}

Note that the method we really want to invoke is `sayGreeting`. Update the
route at line 14 as follows:

{{< highlight kotlin "linenos=table, linenostart=14, hl_lines=2" >}}
            from("direct:greet")
                .bean(HelloBean::class.java, "sayGreeting")   // add processor
                .to("stream:out")
{{</ highlight >}}

Camel will instantiate the bean with the Java class `HelloBean` and adds that
to its registry. The second argument `"sayGreeting"` tells Camel the method
it should invoke.

To run this, add `camel-bean` as a dependency:

{{< highlight kotlin "hl_lines=4" >}}
dependencies {
    // other dependencies are omitted for brevity

    for (s in listOf("core-engine", "direct", "stream", "bean")) {
      implementation("org.apache.camel:camel-${s}:3.1.0")
    }

    implementation("org.slf4j:slf4j-nop:1.7.30")  // so logs won't obfuscate stdout
}
{{</ highlight >}}

And as expected, it prints `Hello, Shaolang` to standard output.

`HelloBean` is an example of a very simple data transformation processor (it
prefix the given string with `Hello, `. Camel has many useful data
transformation components, such as transforming CSV to XML (data format
transformation), and `java.lang.String` to `javax.jms.TextMessage` (data
type transformation). Such transformation processors implements the Message
Translator enterprise integration pattern.

[ei]: https://en.wikipedia.org/wiki/Enterprise_integration
[camel]: https://camel.apache.org
[cia]: https://manning.com/books/camel-in-action-second-edition
[direct-comp]: https://camel.apache.org/components/latest/direct-component.html
[stream-comp]: https://camel.apache.org/components/latest/stream-component.html
