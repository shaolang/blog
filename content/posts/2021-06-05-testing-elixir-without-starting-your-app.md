---
title: "Testing Elixir Without Starting Your App"
date: 2021-06-06T00:08:00+08:00
allowComments: true
tags: elixir
---

By default, [`mix test`][mix-test] starts the project's applications as
part of the task. While this usually is helpful, sometimes not starting
your application is necessary, e.g., when your application's supervisor
starts up singletons that may interfere testing.

Luckily, `mix test` supports the [command line option `--no-start`][clo] to
skip starting the applications after compilation. To make this the "default"
behavior, create the `test` alias to "override" the original in `mix.exs`:

```elixir {linenos=table, hl_lines=[7,14]}
defmodule Sample.MixProject do
  use Mix.Project

  def project do
    [
      app: :sample,
      aliases: aliases(),
      # ... rest omitted for brevity ...
    ]
  end

  defp aliases do
    [
      test: "test --no-start"
    ]
  end
end
```

Doing this alone is not enough: you still need to start the applications that
yours rely on, e.g., the standard library, Elixir itself. To find out the
applications yours depends on, retrieve the list with
[`Application.spec/2`][app-spec] and start them in `test_helper.exs`:

```elixir {linenos=table}
# in test_helper.exs

for app <- Application.spec(:sample, :applications) do
  Application.ensure_all_started(app)
end

ExUnit.start()
```


[mix-test]: https://hexdocs.pm/mix/Mix.Tasks.Test.html
[clo]: https://hexdocs.pm/mix/Mix.Tasks.Test.html#module-command-line-options
[app-spec]: https://hexdocs.pm/elixir/Application.html#spec/2
