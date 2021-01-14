---
title: "SQLite3: Use Nth Value in First Record of Selected Groups"
date: 2021-01-14T14:09:53+08:00
draft: true
allowComments: true
tags: [sql, sqlite3]
---

In a [previous post][prev-post] where I used pandas to transform the following
(with some amendments to the column names):

```html
id | book_id |      bdate | is_borrowed
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

to:

```html
id | book_id |      bdate | is_borrowed | return_date
 0 |     abc | 2020-01-01 |        True |  2020-03-03
 1 |     abc | 2020-02-02 |        True |        NULL
 2 |     abc | 2020-03-03 |        True |        NULL
 3 |     def | 2020-04-04 |       False |        NULL
 4 |     def | 2020-05-05 |       False |        NULL
 5 |     ghi | 2020-06-06 |        True |  2020-08-08
 6 |     ghi | 2020-07-07 |        True |        NULL
 7 |     ghi | 2020-08-08 |        True |        NULL
 8 |     jkl | 2020-09-09 |       False |        NULL
 9 |     jkl | 2020-10-10 |       False |        NULL
10 |     jkl | 2020-11-11 |       False |        NULL
```

SQL (or at least SQLite3 dialect) can achieve the same using
[Common Table Expression][cte] and [SQL window functions][winfn]:

```sql {linenos=table,hl_lines=[2,3]}
WITH data as (
    SELECT MIN(id) OVER (PARTITION BY book_id) AS bid,
           MAX(bdate) OVER (PARTITION by book_id) AS ldate
    FROM books
    WHERE is_borrowed = 'True')
UPDATE books
SET return_date = (SELECT ldate FROM data WHERE books.id = data.bid)
WHERE id IN (SELECT bid FROM data);
```

Within the CTE, the two selected columns make use of window functions; in this
case, the window functions are the familiar aggregate functions `MIN` and `MAX`.
`MIN(id) OVER (PARTITION BY book_id)` returns the earliest `id` by `book_id`,
and `MAX(bdate) OVER (PARTITION BY book_id)` returns the last date also
by `book_id`. Running just the select statement in the CTE returns the
following:

```
bid  ldate
---  ----------
0    2020-03-03
0    2020-03-03
0    2020-03-03
5    2020-08-08
5    2020-08-08
5    2020-08-08
```

The `UPDATE` statement then proceeds to update the table by setting the
respective `return_date`, achieving what the [previous post][prev-post]
did using pandas. One difference though: SQLite3's solution doesn't require
the same number of records for each book; it'll work perfectly fine if
certain books only have one record, or ten, etc.

[prev-post]: ../2020-06-13-pandas-use-nth-value-in-first-rec-of-selected-groups/
[cte]: https://www.essentialsql.com/introduction-common-table-expressions-ctes/
[winfn]: https://www.essentialsql.com/sql-window-functions/
