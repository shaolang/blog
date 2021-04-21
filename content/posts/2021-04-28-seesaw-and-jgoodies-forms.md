---
title: "Seesaw and JGoodies Forms"
date: 2021-04-21T12:36:59+08:00
draft: true
allowComments: true
tags: [clojure, swing]
---

[Seesaw][seesaw] ships [JGoodies Forms][jgoodies-forms] as one of the layout
managers. When creating a `seesaw.forms/forms-panel`, it requires specifying
the `column-spec` for laying out the components in each defined column.
This post covers the `column-spec` that I forget every now and then on
how it works :loudly_crying_face:

`column-spec` is a string of comma-separated specification; the table below
shows the valid specifications:

Spec | Description
-----|------------------------------------------------------------------
pref | Use the component's preferred width
50px | Set column width as 50 pixels
8dlu | Set column width as 8 dialog units

Dialog units is a relative measurement based on the user's preferred font size,
similar to [CSS relative length units][css-units]. An average character is 4
dialog units wide and 8 dialog units high. The image below copied from
[a Stack Overflow question][so-dlu] shows the absolute lengths and widths of
the same 4-by-8 dialog-unit character but for different fonts. The same
question also stated that horizontal and vertical dialog units have
different sizes (also evident in the image below).

![](/images/2021-04-26-dlu-on-different-fonts.png "DLU on different fonts")

Similar to CSS relative length units, using dialog units makes the design
more responsive to layout changes, font usage, DPIs, etc.


[seesaw]: https://github.com/clj-commons/seesaw
[jgoodies-forms]: http://www.jgoodies.com/freeware/libraries/forms/
[css-units]: https://developer.mozilla.org/en-US/docs/Learn/CSS/Building_blocks/Values_and_units#relative_length_units
[so-dlu]: https://stackoverflow.com/questions/395195/wpf-how-to-specify-units-in-dialog-units
