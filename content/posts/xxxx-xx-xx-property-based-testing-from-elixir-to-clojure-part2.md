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

```clojure {linnos=table}
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

```clojure {lineno=table}
(def gen-non-empty-string
  (let [chars "abcdefghijklmnopqrstuvwxyz0123456789"]
    (gen/let [cs (gen/list (gen/elements chars))]
      (apply str (rand-nth chars) cs))))

(def gen-price-list ;; gens {<item> => <price>}
  (gen/let [price-list (gen/not-empty
                        (gen/list (gen/tuple gen-non-empty-string gen/nat)))]
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
empty string, I ended up coding up one that fit my needs better.

`gen-item-list` generates a list of items by choosing the keys of the given
`price-list` map.[^2] The function then loops through the items, retrieves
the price, and sums them up.

[^1]: Erlang introduced maps only from OTP 17.0 onwards.
[^2]: It's a map of items and corresponding price, so calling it a `price-list`
      problably is not ideal.

[prev]: ../2019-08-10-property-based-testing-from-elixir-to-clojure/
[pbtpee]: https://pragprog.com/book/fhproper/property-based-testing-with-proper-erlang-and-elixir
[bttc]: http://codekata.com/kata/kata09-back-to-the-checkout
[kwl-elixir]: https://elixir-lang.org/getting-started/keywords-and-maps.html#keyword-lists
