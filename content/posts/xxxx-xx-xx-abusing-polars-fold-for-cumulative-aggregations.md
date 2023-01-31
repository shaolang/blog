---
title: "(Ab)using Polars Fold for Cumulative Aggregations"
date: 2023-01-30T20:26:10+08:00
draft: true
allowComments: true
tags: [python, polars]
---

[Polars][polars], like most other data wrangling libraries, comes with
row-wise cumulative aggregations out of the box, e.g.,
[`cumsum` (cumulative sum)][cumsum]; it also comes with
column-wise versions that simplify such derivation code, e.g.,
[`fold`][fold]. However, row-wise cumulative aggregations that
depend on multiple columns require some creative use of `fold`.
Before getting in that, let's understand the basic usage of `fold`
first. Given the following dataframe:

```python {linenos=table}
import polars as pl

df = pl.DataFrame(
    dict(
        id=[1, 1, 2, 2],
        a=[10, 20, 50, 60],
        b=[100, 200, 500, 600],
        c=[1000, 2000, 5000, 6000],
    )
)

# id  | a   | b   | c
# --- | --- | --- | ---
# i64 | i64 | i64 | i64
# ====|=====|=====|=====
# 1   | 10  | 100 | 1000
# 1   | 20  | 200 | 2000
# 2   | 50  | 500 | 5000
# 2   | 40  | 600 | 6000
```

The following shows column-wise `fold`:

```python {linenos=table}
df.with_column(
    pl.fold(
        pl.lit(0),                  # initial value
        lambda acc, xs: acc + xs,   # folding function
        ['a', 'b', 'c']             # columns to (left) fold over
    ).alias('column_sum')
)

# id  | a   | b   | c    | column_sum
# --- | --- | --- | ---  | ---
# i64 | i64 | i64 | i64  | i64
# ====|=====|=====|======|===========
# 1   | 10  | 100 | 1000 | 1110
# 1   | 20  | 200 | 2000 | 2220
# 2   | 50  | 500 | 5000 | 5550
# 2   | 60  | 600 | 6000 | 6660
```

_A more elegant, slightly less verbose way is to use
[`reduce`][reduce] instead of `fold`._

While the above is trivial, `fold`'s power shines when there are
many columns to aggregate, or that the folding function can be as
complicated as necessary. `fold` invokes the given lambda/function
(line 4 in above snippet) as many times as the number of columns
given in line 5, where each time, it gives the accumulated value
and the column. Using the snippet above as an example, `fold`
invokes the given lambda with:

* A [`Series`][series] of zeros as the `acc` (accumulated)
  and a `Series` of values from column (aka series) `a` at its
  first invocation.
* The accumulated values from the previous step as `acc`
  and values from column `b` at its second invocation.
* The accumulated values from the previous step as `acc`
  and values from `c` at its last invocation.

[polars]: https://pola.rs
[cumsum]: https://pola-rs.github.io/polars/py-polars/html/reference/expressions/api/polars.Expr.cumsum.html
[fold]: https://pola-rs.github.io/polars/py-polars/html/reference/expressions/api/polars.fold.html
[reduce]: https://pola-rs.github.io/polars/py-polars/html/reference/expressions/api/polars.reduce.html
[series]: https://pola-rs.github.io/polars/py-polars/html/reference/series/index.html
