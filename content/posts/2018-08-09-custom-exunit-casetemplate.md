---
title: "Custom ExUnit.CaseTemplate"
date: 2018-08-09T08:57:56+08:00
tags: [elixir]
---

Creating custom `ExUnit.Case` is done simply by creating a module that
[`use ExUnit.CaseTemplate`][case-template-doc]. However, doing just that is
not enough: when I run `mix test`, the terminal shows the following:

```bash
$ mix test
Compiling 1 file (.ex)
Generated dna app

== Compilation error in file test/my_test.exs ==
** (CompileError) test/my_test.exs:2: module MyCase is not loaded and could not be found
    (elixir) expanding macro: Kernel.use/1
    test/my_test.exs:2: MyTest (module)
    (elixir) lib/code.ex:767: Code.require_file/2
    (elixir) lib/kernel/parallel_compiler.ex:209: anonymous fn/4 in Kernel.ParallelCompiler.spawn_workers/6
```

_(Yeah, I know, I admit: I'm a noob)._

The missing piece is configuring Mix. `mix test` is a task that _only_ runs
tests; it does not compile _and_ run tests. Because the custom case
is a module[^1], Mix needs to know it has to include another directory
when compiling tests:

{{< highlight erlang "hl_lines=8 12-13" >}}
defmodule My.MixProject do
  use Mix.Project

  def project do
    [
      app: :my,
      version: "0.1.0",
      elixirc_paths: elixirc_paths(Mix.env())
    ]
  end

  defp elixirc_paths(:test), do: ["lib", "test"]
  defp elixirc_paths(_),     do: ["lib"]
end
{{</ highlight >}}

[`:elixirc_paths` defaults to `"lib"`][elixirc-paths-default], hence, the need
to make changes similar to the highlighted lines to make custom cases work.


[^1]: The custom case module must end with a `.ex` extension, not `.exs`.

[case-template-doc]: https://hexdocs.pm/ex_unit/ExUnit.CaseTemplate.html#content
[elixirc-paths-default]: https://hexdocs.pm/mix/Mix.Tasks.Compile.Elixir.html#module-configuration
