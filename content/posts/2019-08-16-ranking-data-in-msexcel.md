---
title: Ranking Data in MSExcel
date: 2019-08-16T21:25:48+08:00
allowComments: true
---

MSExcel, despite its [bad name] for
[data analysis], can still be a very good tool to reach out
to for one-off analysis jobs, especially when the output needs to be
interactive. After all, nothing comes close to the reactivity -- as in
[reactive programming] -- simple MSExcel formulae provide
that do not require hardcore development skills.[^1]

## Getting our hands dirty

Let's say the requirement is to rank the customers'  *YTD Trade Count*
and *MTD Trade Count*. One method is to do that by hand:

&nbsp;| A            | B                   | C
------|--------------|:-------------------:|:---------------:
**1** | **Customer** | **YTD Trade Count** | **MTD Trade Count**
**2** | ABC          | 190                 | 7
**3** | DEF          | 288                 | 38
**4** | GHI          | 69                  | 32
**5** | JKL          | 168                 | 38
**6** |
**7** |
**8** |
**9** | **Customer** | **YTD Rank**        | **MTD Rank**
**10**| ABC          | 2nd                 | 4th
**10**| DEF          | 1st                 | 1st
**11**| GHI          | 4th                 | 3rd
**12**| JKL          | 3rd                 | 1st

Yeah, it works! But if the number of customers is in the hundreds, doing it
by hand is a huge waste of time. We can do better than that.

MSExcel has one handy function `LARGE`[^2] which the
[official documentation][large-doc] says:

>> Returns the k-th largest value in a data set.
>> You can use this function to select a value based on its relative standing.
>> For example, you can use LARGE to return the highest, runner-up, or
>> third-place score.

and its syntax:

>> LARGE(array, k)
>>
>> The LARGE function syntax has the following arguments:
>>
>> * Array    Required. The array or range of data for which you want to determine the k-th largest value.
>> * K    Required. The position (from the largest) in the array or cell range of data to return.


Great! Type out the formula `=LARGE(B$2:B$5,ROW()-9)`, copy and paste it
across the range `B10:C13`, and we'll get:

&nbsp;| A            | B                        | C
------|--------------|:------------------------:|:-------------------:
**1** | **Customer** | **YTD Trade Count**      | **MTD Trade Count**
**2** | ABC          | 190                      | 7
**3** | DEF          | 288                      | 38
**4** | GHI          | 69                       | 32
**5** | JKL          | 168                      | 38
**6** |
**7** |
**8** |
**9** | **Customer** | **YTD Rank**             | **MTD Rank**
**10**| ABC          | 2nd                      | 4th
**10**| DEF          | 1st                      | 1st
**11**| GHI          | 4th                      | 3rd
**12**| JKL          | 3rd                      | 1st
**13**|
**14**|
**15**|
**16**|              | **YTD Ordered**          | **MTD Ordered**
**17**|              | =LARGE(B$2:B$5,ROW()-16) | =LARGE(C$2:C$5,ROW()-16)
**18**|              | =LARGE(B$2:B$5,ROW()-16) | =LARGE(C$2:C$5,ROW()-16)
**19**|              | =LARGE(B$2:B$5,ROW()-16) | =LARGE(C$2:C$5,ROW()-16)
**20**|              | =LARGE(B$2:B$5,ROW()-16) | =LARGE(C$2:C$5,ROW()-16)

And that yields the following:

&nbsp;| A            | B                   | C
------|--------------|:-------------------:|:---------------:
**1** | **Customer** | **YTD Trade Count** | **MTD Trade Count**
**2** | ABC          | 190                 | 7
**3** | DEF          | 288                 | 38
**4** | GHI          | 69                  | 32
**5** | JKL          | 168                 | 38
&nbsp;|
&nbsp;| ...          | ...                 | ...
&nbsp;|
**16**|              | **YTD Ordered**     | **MTD Ordered**
**17**|              | 288                 | 38
**18**|              | 190                 | 38
**19**|              | 168                 | 32
**20**|              | 69                  | 7

`ROW()` returns the row number of the cell the function is applied to,
thus `ROW()-16` at `B17` signals to `LARGE` to return the first largest value;
at `B18` to return second largest...

Although that simply sorts the two columns in descending order, it beats
the manual steps in the following ways:

1. Typing out the formula, copy and paste takes only 2 steps, but manual
   sorting takes 4:
   1. Copy values from `B2:B5` and paste them at `B17:B20`
   2. Sort values in `B17:B20`
   3. Copy values from `C2:C5` and paste them at `C17:C20`
   4. Sort values in `C17:C20`
2. MSExcel automatically apply the ordering when any values in `B2:C5` change

Type out the rank at column D (actually, for ranks 4th and beyond, using
the formula `=ROW()-15&"th"` can save some typing with some copy & paste
laziness):

&nbsp;| A            | B                   | C               | D
------|--------------|:-------------------:|:---------------:|:--------:
**16**|              | **YTD Ordered**     | **MTD Ordered** | **Rank**
**17**|              | 288                 | 38              | 1st
**18**|              | 190                 | 38              | 2nd
**19**|              | 168                 | 32              | 3rd
**20**|              | 69                  | 7               | 4th

Replace the hand-coded rank at cell `B10` with the formula
`=VLOOKUP(B2,B$17:$D$17,5-COLUMN(),FALSE)`, then copy and paste the cell across
`B10:C12` yields the same result to the do-by-hand version but with an
important advantage: any changes to the values in `B2:C5` automatically
refresh the ranks.

`COLUMN` returns the column number the function is applied to.
The `5-COLUMN()` fragment in the `VLOOKUP` formula
is to determine automatically the lookup index. For example, at cell `B10`,
`5-COLUMN()` resolves to 3, at cell C10 resolves to 2.

![auto-updated-rankings](/images/2019-08-16-ranking-demo.apng "Auto-updated Ranking")

## Wrapping up

With no coding, some creative use of MSExcel functions, and lots of
copy-and-paste, MSExcel's efficiency is unbeatable. Yes, using [pandas] or
[R] makes the work reproducible, but sometimes not everything is a nail.
MSExcel has its place in data analysis; knowing what tool to use appropriately
is far more important and flexing the scripting muscles.


[^1]: How many data analysts understand reactive programming? My unscientific survey says "not many."
[^2]: The counterpart to `LARGE` is [`SMALL`](https://support.office.com/en-us/article/SMALL-function-17DA8222-7C82-42B2-961B-14C45384DF07)

[bad name]: https://www.google.com/search?hl=en&q=excel%20bad%20for%20data%20analysis
[data analysis]: https://en.wikipedia.org/wiki/Data_analysis
[reactive programming]: https://en.wikipedia.org/wiki/Reactive_programming
[large-doc]: https://support.office.com/en-us/article/LARGE-function-3AF0AF19-1190-42BB-BB8B-01672EC00A64
[pandas]: https://pandas.pydata.org
[R]: https://www.r-project.org
