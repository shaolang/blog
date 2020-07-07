---
title: "Property-Based Testing: From Erlang/Elixir to Clojure Part 2"
date: 2020-07-06T07:32:46+08:00
allowComments: true
draft: true
---

In the [previous post][prev] that covered chapter 5 of
[Property-Based Testing with PropEr, Erlang, and Elixir][pbtpee], I ported
most of the code from the book with very few modifications. Code in
this post that covers chapter 6: Properties-Driven Development are not
straight ports. All the code done [so far] are hosted
at <https://github.com/shaolang/pbtic> I'm using [test.check][test.check]
as the property-based testing tool in Clojure.

In this chapter (of the book), it tackles the [Back to the Checkout][bttc]
code kata: calculate the total price of the items scanned at the checkout
and handle specials correctly.

But first the namespace and relevant `:require`:

```clojure {linenos=table}
(ns pbtic.checkout-test
  (:require [clojure.test.check.clojure-test :refer [defspec]]
            [clojure.test.check.generators :as gen]
            [clojure.test.check.properties :refer [for-all]]
            [pbtic.checkout :as checkout]))
```

## Nothing special

The first property the book implements is "no specials," specifically the
special's price list is empty:

```clojure {linenos=table}
(defspec sums-without-specials
  (for-all [{:keys [items expected-price prices]} gen-item-price-list]
    (= expected-price
       (checkout/total items prices {}))))
```

Simple enough; `gen-item-price-list` generates a map with three items: the
items being checked out, the total expected price, and the price list to use.
Let's take a look at `gen-item-price-list`:

```clojure {linenos=table}
(def gen-item-price-list
  (gen/let [prices                  gen-price-list
            [items expected-price]  (gen-item-list prices)]
    {:items           items
     :expected-price  expected-price
     :prices          prices}))
```

The generated price list `prices` is given to `gen-item-list` to generate
the list of checked out items and calculate the total price. Because there're
no specials, calculating the total price is relatively easier:

```clojure {linenos=table, hl_lines=[14]}
(def gen-non-empty-string
  (let [chars "abcdefghijklmnopqrstuvwxyz0123456789"]
    (gen/let [cs (gen/list (gen/elements chars))]
      (apply str (rand-nth chars) cs))))


(def gen-price-list ;; gens {<item> => <price>}
  (gen/let [price-list (gen/not-empty
                        (gen/list (gen/tuple gen-non-empty-string gen/int)))]
    (into {} price-list)))


(defn gen-item-list [price-list]  ;; gens [[<item>s] {<item> => <price>}]
  (gen/let [items (gen/list (gen/elements (keys price-list)))]
    [items
     (->> items
          (map #(get price-list %))
          (reduce + 0))]))
```

`gen-price-list` matches quite closely to book's version `price_list`
generator, except that the Clojure version returns a map instead of a list.
For this instance, returning a map has two benefits: duplicates are
automatically handled and maps are idiomatic in Clojure (just like [keyword
lists][kwl-elixir] are in Erlang[^1]). As with the book's version,
`gen-price-list` returns a non-empty price list.

`gen-non-empty-string` is a little controversial: I could have used
`clojure.test.check.generators/string-alphanumeric` instead of rolling
my own. However, `clojure.test.check.generators/string-alphanumeric` includes
`""` in its generation. While I could put in checks to ensure there're no
empty string, I ended up coding up one that fits my needs better.

`gen-item-list` generates a list of items by choosing the keys of the given
`price-list` map.[^2] The function then loops through the items, retrieves
the price, and sums them up. Because line 14 does not use `not-empty`
generator, the generated item list may be empty, thus making it return zero
as the expected price.

The simplest working code that satisfies this property looks as follows:

```clojure {linenos=table}
(ns pbtic.checkout)


(defn total [item-list price-list _specials]
  (apply + (map #(get price-list %) item-list)))
```

The actual implementation is a little anti-climax, after all the effort in
writing the generators. But the work done so far covered the following
cases:

* Checkout list is empty
* Order of scanning checkout items are random, i.e., identical items aren't
  necessarily being checked out at the same time
* Specials list is empty

However, the property also implicitly assumes all items being checked out
have the corresponding price in the price list. There's also one (small)
problem with `gen-price-list` that the book addressed as it goes deeper into
this example.[^3]

## Handling Specials

Not wanting to upset the property that's already working well in checking
non-special items, the book advises on creating a new property for handling
specials so that failures when handling specials are due to specials.

```clojure {linenos=table}
(defspec sums-with-specials
  (for-all [{:keys [items expected-price prices specials]} gen-item-price-special]
    (= expected-price
       (checkout/total items prices specials))))
```

The property itself looks simple; it's largely the same as the regular one
above, except that it uses `gen-item-price-special` that also generate
specials:

```clojure {linenos=table, hl_lines=[4,5]}
(def gen-item-price-special
  (gen/let [prices    gen-price-list
            specials  (gen-special-list prices)
            [spec-items spec-price] (gen-special prices specials)
            [reg-items reg-price]   (gen-regular prices specials)]
    {:items           (shuffle (concat spec-items reg-items))
     :expected-price  (+ spec-price reg-price)
     :prices          prices
     :specials        specials}))
```

The cleverness of this generator[^4] is that it separates the generating of
the set of items that always gets the special prices (line 4, using
`gen-special`) and the generating that never gets the special prices (line 5,
using `gen-regular`). The alternative is to generate all the items,
sieves through the generated list, determines the items that get special
prices, determines the remaining with regular prices, and sums that total.
That alternative is actually the "production" code; we should not "replicate"
such code in test code!

```clojure {linenos=table}
(defn gen-special-list [prices] ;; gens {<item> => {:count <n> :price <n>}}
  (gen/let [specials (gen/list (gen/tuple (gen/elements (keys prices))
                                          (gen/choose 2 5)
                                          gen/int))]
    (apply merge
           (map (fn [[item count price]] {item {:count count :price price}})
                specials))))
```

`gen-special-list` is simple enough; it generates a list of tuples where
each tuple contains the item (name), a quantity (item count to qualify for
special price), and the price. Its body then transforms the generated list of
tuples into a map of specials. `gen-special-list` has the same (small) issue
as `gen-price-list`, as well as one new issue (which actually isn't that
important and could be left as such; try thinking about it before
turning to the footnote[^5]).

[^1]: Erlang introduced maps only from OTP 17.0 onwards.
[^2]: It's a map of items and corresponding price, so calling it a `price-list`
      problably is not ideal.
[^3]: If I remember correctly, I actually hit the issue and had to address
      it before proceeding on to the specials property. Nevertheless,
      I'm keeping the post as-is to follow the book's flow.
[^4]: Actually, more appropriately, Fred's wisdom.
[^5]: Because the price generated isn't checked against the price list, it's
      possible that the "special" price is actually more costly :joy: Nope,
      the book didn't address this, 'cos it really is a small problem.

[prev]: ../2019-08-10-property-based-testing-from-elixir-to-clojure/
[pbtpee]: https://pragprog.com/book/fhproper/property-based-testing-with-proper-erlang-and-elixir
[bttc]: http://codekata.com/kata/kata09-back-to-the-checkout
[kwl-elixir]: https://elixir-lang.org/getting-started/keywords-and-maps.html#keyword-lists
