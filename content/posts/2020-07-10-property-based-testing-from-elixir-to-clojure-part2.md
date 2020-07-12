---
title: "Property-Based Testing: From Erlang/Elixir to Clojure Part 2"
date: 2020-07-10T22:00:00+08:00
allowComments: true
tags: [clojure, property-based testing]
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

## Nothing Special

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

What's left from `gen-item-price-special` are `gen-special` and `gen-regular`;
due to Clojure's lack of pattern matching at the function level, the code
looks a little different. First, `gen-regular`:

```clojure {linenos=table}
(defn gen-regular [prices specials]
  (letfn [(gen-count [item]
            (if (contains? specials item)
              (gen/choose 0 (dec (get-in specials [item :count])))
              gen/nat))]
    (gen/return
      (->> prices
           (map (fn [[item price]]
                  (let [counts (gen/generate (gen-count item))]
                    {:items     (repeat counts item)
                     :sub-total (* counts price)})))
           (reduce (fn [{:keys [all-items total] :as result}
                        {:keys [items sub-total]}]
                     (assoc result
                            :total (+ total sub-total)
                            :all-items (shuffle (concat all-items items))))
                   {:total 0
                    :all-items []})
           ((juxt :all-items :total))))))
```

`gen-regular` defines an inline function `gen-count` that restricts the
generation of item count depending on whether the item exists in the
special list or not:

* If it does, it generates a number between zero and one-less than the
  special's count, i.e., it always generates a number that never triggers
  the special price to kick in.
* Otherwise, it generates any number of items, even zero.

Then, for each item in the price list, it generates the item count using
the inline `gen-count` function (`map` operation from lines 8 to 11),
calculates the total price and creates the item list (`reduce` operation
from lines 12 to 18), and returns the total price and item list (line 19).
The shuffling of the items at line 16 is to keep to the "randomness" when
checking out, i.e., same items aren't always checked out together.

```clojure {linenos=table}
(defn gen-special [_ specials]
  (gen/let [multipliers (gen/vector gen/nat (count specials))]
    (->> (zipmap specials multipliers)
         (map (fn [[[item {:keys [count price]}] multiplier]]
                {:items     (repeat (* count multiplier) item)
                 :sub-total (* price multiplier)}))
         (reduce (fn [{:keys [all-items total] :as result}
                      {:keys [items sub-total]}]
                   (assoc result
                          :all-items (concat all-items items)
                          :total     (+ total sub-total)))
                 {:total 0
                  :all-items []})
         ((juxt :all-items :total)))))
```

`gen-special` is relatively similar in its map-reduce operation; the main
difference lies in it generating multipliers--which could be zero--for each
special that is used to create the item list based on these multipliers.
Yeah, the destructuring at line 4 is a little involved, but that makes the
anonymous function's body cleaner.

The updated `pbtic.checkout/total` that handles specials as well looks as
follows:

```clojure {linenos=table}
(defn total [item-list price-list specials]
  (let [counts (count-seen item-list)
        {:keys [counts-left prices]} (apply-specials counts specials)]
    (+ prices (apply-regular counts-left price-list))))
```

`pbtic.checkout/total` uses three helper functions:

* `count-seen`: returns a map of items and the item counts based on the
  given item list; it essentially just aliases to the built-in
  `frequencies` function
* `apply-specials`: returns a map of remaining items that do not qualify
  for special prices and the total price of items that qualify
* `apply-regular`: returns total prices of remaining items

```clojure {linenos=table}
(def count-seen frequencies)


(defn apply-specials [counts specials]
  (reduce (fn [{:keys [prices counts-left]} [item seen]]
            (if-let [{:keys [count price]} (get specials item)]
              {:counts-left (merge counts-left {item (rem seen count)})
               :prices      (+ prices (* (quot seen count) price))}
              {:counts-left (merge counts-left {item seen})
               :prices      prices}))
          {:counts-left nil :prices 0}
          counts))


(defn apply-regular [items prices]
  (transduce (map (fn [[item count]] (* count (get prices item))))
             +
             0
             items))
```

## Negative Testing
Code written so far covers the happy path: it works when inputs are correct.
Negative testing covers the path less travelled. Writing broad properties
that test the more general properties of the code can help in checking
whether the happy-path properties are consistent. The broader such properties
are, the better they are in searching for problems that we expect.[^6]
Think of broad properties as supplements or anchors to the specific ones, they
aren't too useful on their own.

```clojure {linenos=table}
(defspec negative-testing-for-expected-results
  (for-all [{:keys [items prices specials]} gen-lax-lists]
    (integer? (checkout/total items prices specials))))
```

This property is very, very broad: it expects `pbtic.checkout/total` always
return an integer. But instead of reusing existing generators, it uses a new
one `gen-lax-lists`:

```clojure {linenos=table}
(def gen-lax-lists
  (gen/hash-map
   :items     (gen/list gen/string-alphanumeric)
   :prices    (gen/fmap #(into {} %)
                        (gen/list (gen/tuple gen/string-alphanumeric gen/int)))
   :specials  (gen/fmap (fn [vs]
                          (->> vs
                               (map (fn [[item count price]]
                                      {item {:count count :price price}}))
                               (apply merge)))
                        (gen/list (gen/tuple gen/string-alphanumeric
                                             gen/int
                                             gen/int)))))
```

Unlike `gen-item-price-list` and `gen-item-price-special`, `gen-lax-lists`
does not generate the checkout items and specials from its generated price list.
When we run the test, we'll be greeted with a glorious stacktrace caused by
`java.lang.NullPointerException`. Let's make the test output friendlier by
wrapping the body in try-catch:

```clojure {linenos=table}
(defspec negative-testing-for-expected-results
  (for-all [{:keys [items prices specials]} gen-lax-lists]
    (try
      (integer? (checkout/total items prices specials))
      (catch Throwable _ false))))
```

Running the above, you'll see something similar to the following:

```
FAIL in (negative-testing-for-expected-results) (checkout_test.clj)
expected: {:result true}
  actual: {:shrunk
           {:total-nodes-visited 29,
            ;; omitted for brevity
            :smallest
            [{:items (""),
              :prices {},
              :specials nil}]},
             ;; omitted for brevity
          }
```

Looking up the price of an unknown item causes the problem. This probably
is acceptable but instead of throwing a generic `NullPointerException`,
throwing one with more information probably is more helpful. Adjusting
the property:

```clojure {linenos=table}
(defspec negative-testing-for-expected-results
  (for-all [{:keys [items prices specials]} gen-lax-lists]
    (try
      (integer? (checkout/total items prices specials))
      (catch Throwable ex
        (contains (ex-data e) :unknown-item)))))
```

And updating the code in `pbtic.checkout`:

```clojure {linenos=table, hl_lines=[8]}
(defn cost-of-item [prices item]
  (if-let [price (get prices item)]
    price
    (throw (ex-info "unknown-item" {:unknown-item item}))))


(defn apply-regular [items prices]
  (transduce (map (fn [[item count]] (* count (cost-of-item prices item))))
             +
             0
             items))
```

The new `cost-of-item` function throws the exception with more helpful
information when it can't retrieve the item's prices from the given `prices`
map. `apply-regular` then replaces the original `(get prices item)` with
`(cost-of-item prices item)` to satisfy the property's expected outcome.
Running the test again, another error comes up[^7]:

```
FAIL in (negative-testing-for-expected-results) (checkout_test.clj)
expected: {:result true}
  actual: {:shrunk
           {:total-nodes-visited 29,
            ;; omitted for brevity
            :smallest
            [{:items ("C"),
              :prices {},
              :specials {"C" {:count 0, :price 1}}}]},
             ;; omitted for brevity
          }
```

A very lax specials; let's adopt the same strategy in throwing a more helpful
exception:

```clojure {linenos=table, hl_lines=[6, 7]}
(defspec negative-testing-for-expected-results
  (for-all [{:keys [items prices specials]} gen-lax-lists]
    (try
      (integer? (checkout/total items prices specials))
      (catch Throwable ex
        (or (contains (ex-data e) :unknown-item)
            (contains (ex-data e) :invalid-special-list)))))
```

And changes in `pbtic.checkout`:

```clojure {linenos=table, hl_lines=[6, 7]}
(defn valid-special-list? [specials]
  (every? #(pos? (:count %)) (vals specials)))


(defn total [item-list price-list specials]
  (when-not (valid-special-list? specials)
    (throw (ex-info "invalid special list" {:invalid-special-list true})))
  (let [counts (count-seen item-list)
        {:keys [counts-left prices]} (apply-specials counts specials)]
    (+ prices (apply-regular counts-left price-list))))
```

Unfortunately, `test.check` doesn't have any functions to collect statistics
on the generated values, so we'll just continue with text in the book and
make `gen-lax-lists` a little stricter so that the property is closer
to the "nothing works" end of the spectrum[^8]:

```clojure {linenos=table}
(def gen-lax-lists
  (let [known-items ["A", "B", "C"]
        gen-maybe-known (gen/bind (gen/list gen-non-empty-string)
                                  #(gen/elements (concat known-items %)))]
    (gen/hash-map
     :items   (gen/list gen-maybe-known)
     :prices  (gen/fmap #(into {} %) (gen/list (gen/tuple gen-maybe-known
                                                          gen/nat)))
     :specials (gen/fmap (fn [vs]
                           (->> vs
                                (map (fn [[item count price]]
                                       {item {:count count :price price}}))
                                (apply merge)))
                         (gen/list (gen/tuple gen-maybe-known
                                              gen/nat
                                              gen/nat))))))
```

Unlike the book's code, the above does not cause any new failures in/expose
any issues with the code written so far. In fact, because we adopted Clojure's
idiom in using maps, instead of keyword lists, the other issues mentioned
in the book are not applicable to our code so far.

## Summary

This wraps up the porting of Erlang/Elixir code from chapter 6 to Clojure.
In the next post, I'll cover stateful properties.

[^1]: Erlang introduced maps only from OTP 17.0 onwards.
[^2]: It's a map of items and corresponding price, so calling it a `price-list`
      problably is not ideal.
[^3]: If I remember correctly, I actually hit the issue at this point in
      time (for both PropCheck and test.check) and had to address
      it before proceeding on to the specials property. Nevertheless,
      I'm keeping the post as-is to follow the book's flow.
[^4]: Actually, more appropriately, Fred's wisdom.
[^5]: Because the price generated isn't checked against the price list, it's
      possible that the "special" price is actually more costly :joy:
[^6]: Finding cases that we didn't think of is exactly the reason why
      we adopt property-based testing.
[^7]: The book did not hit this problem at this point in time; it hit this
      much later on.
[^8]: Buy the book to understand what this means. Alternatively,
      you can read the
      [description at the book's old-but-free website][checkout]

[prev]: ../2019-08-10-property-based-testing-from-elixir-to-clojure/
[pbtpee]: https://pragprog.com/book/fhproper/property-based-testing-with-proper-erlang-and-elixir
[test.check]: https://github.com/clojure/test.check
[bttc]: http://codekata.com/kata/kata09-back-to-the-checkout
[kwl-elixir]: https://elixir-lang.org/getting-started/keywords-and-maps.html#keyword-lists
[checkout]: https://propertesting.com/book_case_study_stateless_properties_with_a_checkout.html#_negative_tests
