---
title: "SQLite: show UTC timestamp as local time"
date: 2019-06-24T21:12:03+08:00
tags: [sqlite3]
---

A business unit asked me to compile some trading volume stats by trading hour.
The good thing is that I have the following in SQLite database; the bad thing
is the database stores the timestamp as follows:

<table>
  <thead>
    <tr><th>trade id</th><th>amount done</th><th>timestamp</th></tr>
  </thead>
  <tbody>
    <tr><td>1</td><td>100</td><td>2019-06-01 01:01:01.111111</td></tr>
    <tr><td>2</td><td>200</td><td>2019-06-01 02:02:02.222222</td></tr>
    <tr><td>3</td><td>300</td><td>2019-06-01 03:03:03.333333</td></tr>
    <tr><td colspan='3' style='text-align: center;'>...</td>
    <tr><td>1000</td><td>10000</td><td>2019-06-24 23:59:59.999999</td></tr>
  </tbody>
</table>

`timestamp` column is stored as a string[^1], so rather than doing timezone
arithmetic, I used SQLite's built-in date/time functions instead:

```
WITH
epochized AS (
    SELECT STRFTIME('%s', [timestamp]) AS [epoch]    -- convert to unix epoch
    FROM trades),
localized AS (
    SELECT TIME([epoch], 'unixepoch', 'localtime')  -- convert epoch to local time
    FROM dropped_offsets),
SELECT SUBSTR([localtime], 1, 2) AS [trading_hour]  -- extract hour
FROM localized;
```

_The above employs [common table expressions][cte] for clarity; the actual
combined them as one instead._

[^1]: Okay, to be fair, SQLite employs what it terms as [type affinity](https://sqlite.org/datatype3.html#type_affinity)

[cte]: https://en.wikipedia.org/wiki/Hierarchical_and_recursive_queries_in_SQL#Common_table_expression
