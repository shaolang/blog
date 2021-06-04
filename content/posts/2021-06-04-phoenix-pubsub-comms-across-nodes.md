---
title: "Phoenix Pubsub Comms Across Nodes"
date: 2021-06-04T22:21:36+08:00
allowComments: true
tags: elixir
---

Phoenix PubSub makes cross-nodes publish-subscribe extremely simple: setting up
one for single node is the same as setting up multiple. This post covers the
steps in doing so.

First, run `mix new pub_sub --sup` to generate a new project. Then add
`{:phoenix_pubsub, "~> 2.0"}` as a dependency in `mix.exs`.
Open `lib/pub_sub/application.ex` to add Phoenix PubSub to the supervision
tree:

```elixir {linenos=table, hl_lines=[8]}
defmodule PubSub.Application do
  @moduledoc false
  use Application

  @impl true
  def start(_type, _args) do
    children = [
      {Phoenix.PubSub, name: PS}      # add this line
    ]

    opts = [
      strategy: :one_for_one,
      name: PubSub.Supervisor
    ]

    Supervisor.start_link(children, opts)
  end
end
```

That's all to it!

Start the application in one terminal, using `node@localhost` as the node's
short name and start interacting with it:

```bash {hl_lines=[11]}
$ iex --sname node@localhost -S mix

iex(node@localhost)1> alias Phoenix.PubSub
iex(node@localhost)2> PubSub.subscribe(PS, "user:123")
iex(node@localhost)3> Process.info(self(), :messages)
{:messages, []}
iex(node@localhost)4> PubSub.broadcast(PS, "user:123", {node(), "hello"})
:ok
iex(node@localhost)5> Process.info(self(), :messages)
{:messages, [node@localhost: "hello"]}
iex(node@localhost)6> flush()
iex(node@localhost)7> Process.info(self(), :messages)
{:messages, []}
```

Before proceeding further, flush the process' message box as shown above.
Then in a separate terminal, start a node `deno@localhost` and connect it
to the previous one:

```bash {hl_lines=[8]}
$ iex --sname deno@localhost -S mix

iex(deno@localhost)1> Node.connect(:node@localhost)
iex(deno@localhost)2> alias Phoenix.PubSub
iex(deno@localhost)4> PubSub.broadcast(PS, "user:123", {node(), "goodbye"})
:ok
iex(deno@localhost)5> Process.info(self(), :messages)
{:messages, []}
```

In this second terminal, notice that its process' message box is empty because
this node doesn't subscribe to the topic `user:123`. Switching back to the
first terminal, query the process' message box to confirm the receipt of
the message broadcast from the second (`deno`) node:

```bash {hl_lines=[2]}
iex(node@localhost)8> Process.info(self(), :messages)
{:messages, [deno@localhost: "goodbye"]}
```

Amazing, right?

A few things to note:

* Every node must use the same name when starting the PubSub service. If the
  name in one node is `PS` and the other is `PS2`, broadcasts won't propagate
  across the node.
* Phoenix PubSub uses Erlang's `pg2` module by default, which OTP 24 will
  remove and replace with `pg` module.
