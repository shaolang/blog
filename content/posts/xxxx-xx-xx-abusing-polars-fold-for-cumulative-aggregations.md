---
title: "(Ab)using Polars Fold for Cumulative Aggregations"
date: 2023-01-30T20:26:10+08:00
draft: true
allowComments: true
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
        id=[100, 100, 200, 200],
        a=[1, 2, 3, 4],
        b=[10, 20, 30, 40],
    )
)

# id  | a   | b
# --- | --- | ---
# i64 | i64 | i64
# ====|=====|====
# 1   | 10  | 100
# 1   | 20  | 200
# 2   | 50  | 500
# 2   | 40  | 600
```

The following shows column-wise `fold`:

```python {linenos=table}
df.with_column(
    pl.fold(
        pl.lit(0),                  # initial value
        lambda acc, xs: acc + xs,   # folding function
        ['a', 'b']                  # columns to (left) fold over
    ).alias('a_plus_b')
)

# id  | a   | b   | a_plus_b
# --- | --- | --- | ---
# i64 | i64 | i64 | i64
# ====|=====|=====|=========
# 1   | 10  | 100 | 110
# 1   | 20  | 200 | 220
# 2   | 50  | 500 | 550
# 2   | 60  | 600 | 660
```

_A more elegant, slightly less verbose way is to use
[`reduce`][reduce] instead of `fold`._

While the above is trivial, `fold`'s power shines when there are
many columns to aggregate, or that the folding function can be as
complicated as necessary.

[polars]: https://pola.rs
[cumsum]: https://pola-rs.github.io/polars/py-polars/html/reference/expressions/api/polars.Expr.cumsum.html
[fold]: https://pola-rs.github.io/polars/py-polars/html/reference/expressions/api/polars.fold.html
[reduce]: https://pola-rs.github.io/polars/py-polars/html/reference/expressions/api/polars.reduce.html
