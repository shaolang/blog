---
title: "Moving Away From Numpy Random Functions"
date: 2021-12-01T09:30:20+08:00
draft: true
allowComments: true
tags: [python, numpy]
---

NumPy's [numpy.random.seed][seed] stated clearly that:

> This is a convenience, legacy function.

And its [Legacy Random Generation][lrg] states that "The [legacy]
[RandomState][random-state] provides access to legacy generators.
This generator is considered frozen and will have no further
improvements."

[seed]: https://numpy.org/doc/stable/reference/random/generated/numpy.random.seed.html
[lrg]: https://numpy.org/doc/stable/reference/random/legacy.html
[random-state]: https://numpy.org/doc/stable/reference/random/legacy.html#numpy.random.RandomState
