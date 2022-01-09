---
title: "Links"
---

List of interesting/helpful/queer/etc links:

## Clojure

- [Clojure, Faster](https://tech.redplanetlabs.com/2020/09/02/clojure-faster): optimizing Clojure when it matters
- [Fast and Elegant Clojure](https://bsless.github.io/fast-and-elegant-clojure/)
- [On The Nature Of Clojure Protocols](https://flexiana.com/2021/08/on-the-nature-of-clojure-protocols)
- [Practical Data Coercion with Prismatic/schema](https://camdez.com/blog/2015/08/27/practical-data-coercion-with-prismatic-schema): especially on how to make multiple coercion matchers work together
- [Taming Advanced Compilation Bugs in ClojureScript Projects](https://dev.solita.fi/2020/06/25/taming-cljs-advanced-compilation.html)

## CSS
- [CSS Tips](https://markodenic.com/css-tips/): CSS tips and tricks you won't see in most of the tutorials
- [Defensive CSS](https://ishadeed.com/article/defensive-css/)

# Elixir

- [Avoiding GenServer bottlenecks](https://www.cogini.com/blog/avoiding-genserver-bottlenecks/)
- [Elixir: a few things about GenStage I wish I knew some time ago](https://medium.com/@andreichernykh/elixir-a-few-things-about-genstage-id-wish-to-knew-some-time-ago-b826ca7d48ba)

# Machine Learning/Artificial Intelligence
- [5 Deep Learning Activation Functions You Need to Know](https://builtin.com/machine-learning/activation-functions-deep-learning)

# Python

- [Profiling and Analyzing Performance of Python Programs](https://martinheinz.dev/blog/64)
- [pytz: The Fastest Footgun in the West](https://blog.ganssle.io/articles/2018/03/pytz-fastest-footgun.html): why dateutil is preferred over pytz

# Rust

- [A Survey of Rust GUI Libraries (as of 21 Aug 2020)](https://www.boringcactus.com/2020/08/21/survey-of-rust-gui-libraries.html)
- [Rust GUI: Introduction, a.k.a. the state of Rust GUI libraries (as of Jan 2021)](https://dev.to/davidedelpapa/rust-gui-introduction-a-k-a-the-state-of-rust-gui-libraries-as-of-january-2021-40gl)

## SQL

- [How do I insert a row which contains a foreign key?](https://dba.stackexchange.com/a/46415): with concise and smart SQL statement to insert many records
- [SQL Style Guide](https://www.sqlstyle.guide)

## Misc

- [AWS services explained in one line each](https://adayinthelifeof.nl/2020/05/20/aws.html)
- [Don't Start With Microservices - Monoliths Are Your Friend](https://arnoldgalovics.com/microservices-in-production/)
- [Framework for writing good documentation](https://documentation.divio.com)
- [Git - When to Merge vs. When to Rebase](https://www.derekgourlay.com/blog/git-when-to-merge-vs-when-to-rebase/)
  - Rule of thumb:
    - When pulling changes from origin/develop (or origin/master) onto local develop, use rebase
    - When finishing a feature branch, merge the changes back to develop (or master)
  - When to use what:
    - `git pull origin` when no local changes
    - `git pull --rebase origin` when there are local changes (but latest commit is not a merged with feature branch)
    - `git merge --no-ff <branch>` when merging feature branch back to master
    - `git fetch origin` then `git rebase -r origin/<branch>` when there are local changes (and last commit is a merge with feature branch)
- [On Coding, Ego, and Attention](https://josebrowne.com/on-coding-ego-and-attention): on self-improvement by not making work personal
- [Prioritize Software Features By Mapping Complexity & Value With a Feature Matrix](https://spin.atomicobject.com/2021/01/27/prioritize-software-features/)
