---
title: "Property-Based Testing: From Erlang/Elixir to Clojure Part 3"
date: 2020-08-03T07:39:50+08:00
draft: true
allowComments: true
tags: [clojure, property-based testing]
---

Moving from stateless testing in the [previous post][prev], this post covers
the fundamental of stateful testing from chapter 9 of the [book][pbtpee]:
run stateful tests on a cache that evicts older items beyond the stated
size and replace existing entry if the key of the entry already exists in
the cache. Such toy examples are typically used to demonstrate the basics
of stateful testing.

## Code Under Test
First, the code that will be put to the stateful test:

```clojure {linenos=table, hl_lines=[23,24]}
(ns pbtic.cache
  (:refer-clojure :exclude [find]))

(def ^:private *ets (atom nil))

(defn init! [max-items]
  (reset! *ets {:index 0 :max-items max-items}))


(defn- find* [k]
  (some (fn [[i vs :as entry]] (when (and (vector? vs) (= k (first vs))) entry))
        @*ets))


(defn find [k]
  (when-let [[_ [_ v]] (find* k)]
    v))


(defn cache! [k v]
  (if-let [[i] (find* k)]
    (swap! *ets assoc i [k v])
    (let [new-index (inc (mod (:index @*ets) (:max-items @*ets)))]
      (swap! *ets assoc new-index [k v] :index new-index))))


(defn flush! []
  (let [{:keys [max-items]} @*ets]
    (init! max-items)))
```

The book uses [Erlang Term Storage (ETS)][ets] as the in-memory table for
storing Erlang [terms][erlang-term] (or simply, data). For simplicity, I'm
using an `atom` that holds a hash-map. I've also purposely made the atom
private to the namespace so that the test code can only access it via the
following four functions:

* `init!`: initializes the atom with two special keys in the hash-map, an
  `index` key to keep track of the current index and `max-items` to limit
  the number of items allowable to be kept in the hash-map
* `flush!`: clears the hash-map and resets the index; notice that `max-items`
  is copied from the hash-map before clearing
* `cache!`: adds the key-value entry to the hash-map if the key is new,
  replaces if the key is existing or when the hash-map has reached max capacity
* `find`: returns the value of the given key, nil if key is non-existent

Each entry `cache!` inserts takes the form `{i [k v]}` where `i` is the
index (as key) and `[k v]` as the value. Such form allows the replacement
of older entries easily (evident at lines 23 and 24).

## Writing Properties
[stateful-check][stateful-check] is stateful property-based testing tool
that is built on top of [test.check][test.check]. First, the namespace and
a few definitions for this property test:

```clojure
(ns pbtic.cache-test
  (:require [clojure.test :refer [deftest is]]
            [clojure.test.check.generators :as gen]
            [pbtic.cache :as c]
            [stateful-check.core :refer [specification-correct?]]))

(def cache-size 10)
(def known-keys [:hello :hi :konichiwa])
(def gen-key (gen/one-of [(gen/elements known-keys) gen/keyword]))
(def gen-val gen/nat)
```

Following the book's direction, this test specifies the cache size for
simplicity, i.e., it's entirely possible to generate the size using
`(gen/fmap inc gen/nat)`.[^1] `known-keys` is a small set of keywords to
help the `gen-key` generator in generating repeated keys to force the
test to exercise code related to replacing and finding values of known
keys.

[^1]: This expression creates a generator that generates positive numbers.


[prev]: ../2020-07-10-property-based-testing-from-elixir-to-clojure-part2/
[pbtpee]: https://pragprog.com/book/fhproper/property-based-testing-with-proper-erlang-and-elixir
[ets]: http://erlang.org/doc/man/ets.html
[erlang-term]: http://erlang.org/doc/reference_manual/data_types.html#term
[stateful-check]: https://github.com/czan/stateful-check
[test.check]: https://github.com/clojure/test.check
