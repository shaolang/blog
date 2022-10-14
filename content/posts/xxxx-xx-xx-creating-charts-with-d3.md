---
title: "Creating Charts With D3.js"
date: 2022-10-11T21:17:09+08:00
draft: true
allowComments: true
tags: [javascript]
---

[D3.js][d3js] is extremely versatile, widely, yet somewhat difficult to use.
It probably is due to my lack of practice/opportunity to use, so I'm going
to leave some notes for my future self. Although recommendations generally
revolve around adapting the examples from D3's website, having a solid
foundation would definitely help in reading codes from the around the web.
We'll use [USDSGD][usdsgd] prices (from Yahoo! Fiance) as the [data to work
with](/usdsgd.csv) for the post:

```csv
Date,Open,High,Low,Close,Adj Close,Volume
2022-08-08,1.382120,1.383070,1.376700,1.382120,1.382120,0
2022-08-09,1.378250,1.379380,1.376400,1.378250,1.378250,0
2022-08-10,1.378300,1.379770,1.367400,1.378300,1.378300,0
2022-08-11,1.368800,1.372620,1.366540,1.368800,1.368800,0
2022-08-12,1.369600,1.372240,1.368600,1.369600,1.369600,0
```

The following snippet shows how `d3.csv()` converts the raw data
into JavaScript data structures:

{{< highlight javascript "linenos=table">}}
import * as d3 from 'd3';

const data = d3.csv('/usdsgd.csv', row => ({
  date: row['Date'],
  close: Number.parseFloat(row['Close']),
}));

console.info(data[0]);  // outputs { date: '2022-01-03', close: '1.3482' }
{{</ highlight >}}

## Building blocks
Similar to most things, charts built with D3 are made up of building
blocks[^1]. Data accessors and scalers are two such building blocks:

* Accessors: retrieve relevant information from loaded domain data
* Scalers: transform domain data into plotting values

The sketch below shows the politically incorrect relationships these have
with the data and <abbr title='Document Object Model'>DOM</abbr>:

```
Data ==> Accessor ==> Scaler ==> DOM
```

When done right, code manipulating the DOM doesn't retrieve values using
the accessors without going through scalers. For example, when plotting
a scatter plot on the outcomes of a die roll, DOM rendering code shouldn't
use the value returned by the accessors directly; DOM rendering code should
use the scalers' outputs because scalers scale the accessors' (returned)
values to DOM units, e.g., pixels.

[d3js]: https://d3js.org
[usdsgd]: https://finance.yahoo.com/quote/SGD%3DX/history?p=SGD%3DX
