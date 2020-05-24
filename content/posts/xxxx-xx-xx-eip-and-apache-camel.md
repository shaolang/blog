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

[^1]: [Camel in Action, 2nd Edition][cia2e] actually did recommend
      reading the EIP book shortly into chapter 1, but I ignored that
      advice :sweat_smile:

[camel]: https://camel.apache.org
[eip]: https://www.enterpriseintegrationpatterns.com
[cia2e]: https://www.manning.com/books/camel-in-action-second-edition
