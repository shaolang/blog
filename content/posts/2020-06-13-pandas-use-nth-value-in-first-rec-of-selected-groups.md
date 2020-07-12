---
title: "pandas: Use Nth Value in First Record of Selected Groups"
date: 2020-06-13T08:12:09+08:00
allowComments: true
tags: [python, pandas]
---

How do you transform the following data frame:

```html
   | book_id |       date | is_borrowed
 0 |     abc | 2020-01-01 |        True
 1 |     abc | 2020-02-02 |        True
 2 |     abc | 2020-03-03 |        True
 3 |     def | 2020-04-04 |       False
 4 |     def | 2020-05-05 |       False
 5 |     ghi | 2020-06-06 |        True
 6 |     ghi | 2020-07-07 |        True
 7 |     ghi | 2020-08-08 |        True
 8 |     jkl | 2020-09-09 |       False
 9 |     jkl | 2020-10-10 |       False
10 |     jkl | 2020-11-11 |       False
```

to the following, i.e., for each book that is borrowed, take the last
date record in the group as the return date?


```html
   | book_id |       date | is_borrowed | return_date
 0 |     abc | 2020-01-01 |        True | 2020-03-03
 1 |     abc | 2020-02-02 |        True |        NaN
 2 |     abc | 2020-03-03 |        True |        NaN
 3 |     def | 2020-04-04 |       False |        NaN
 4 |     def | 2020-05-05 |       False |        NaN
 5 |     ghi | 2020-06-06 |        True | 2020-08-08
 6 |     ghi | 2020-07-07 |        True |        NaN
 7 |     ghi | 2020-08-08 |        True |        NaN
 8 |     jkl | 2020-09-09 |       False |        NaN
 9 |     jkl | 2020-10-10 |       False |        NaN
10 |     jkl | 2020-11-11 |       False |        NaN
```

First (naive) attempt:

```python {linenos=table}
# assume the data is loaded in variable df

df.loc[df.is_borrowed].groupby('book_id').first()['return_date'] = \
    df.loc[df.is_borrowed].groupby('book_id').nth(2)['date']
```

But it doesn't work[^1]: the new column `return_date` isn't shown when
printing `df`.

So I did the following which probably is unorthodox but is simple enough
to grok:

```python {linenos=table}
reps = len(df.loc[df.is_borrowed]) // 3
df.loc[df.is_borrowed, 's_no'] = [1, 2, 3] * reps
df.loc[df.s_no == 1, 'returned_date'] = list(df.loc[df.s_no == 3, 'date'])
df.drop('s_no', axis=1, inplace=True)
```

Let's walk through the code above.

Line 1 is simply counting the number of groups that satisfy the condition
`is_borrowed`. In this example, `reps` is 2 'cos there are two such
groups.

Line 2 populates `s_no` but only for qualified groups. We need to multiply
the array by `reps` because the number of values on the right-hand side of `=`
must be the same as the number of "selected" rows on the left. At this point in
time, `df` looks as follows:

```html
   | book_id |       date | is_borrowed | s_no
 0 |     abc | 2020-01-01 |        True |  1.0
 1 |     abc | 2020-02-02 |        True |  2.0
 2 |     abc | 2020-03-03 |        True |  3.0
 3 |     def | 2020-04-04 |       False |  NaN
 4 |     def | 2020-05-05 |       False |  NaN
 5 |     ghi | 2020-06-06 |        True |  1.0
 6 |     ghi | 2020-07-07 |        True |  2.0
 7 |     ghi | 2020-08-08 |        True |  3.0
 8 |     jkl | 2020-09-09 |       False |  NaN
 9 |     jkl | 2020-10-10 |       False |  NaN
10 |     jkl | 2020-11-11 |       False |  NaN
```

Line 3 then "selects" all first rows in qualified rows and populates
`returned_date` with `date` values from "selected" third rows. Note that
we need to convert the values on the left to a list; otherwise, the
assignment won't work.[^2]

Line 4 simply drops the temporary `s_no`, leaving us with the data frame
we want.

It works, though probably unorthodox. Until I'm more versed with pandas,
this works for me for now.

[^1]: At of pandas 1.0.x, it doesn't output any warning/error messages or
      crash when running this snippet.
[^2]: Don't ask me why; I haven't figured out why the assignment works only
      when the conversion to list occurs.
