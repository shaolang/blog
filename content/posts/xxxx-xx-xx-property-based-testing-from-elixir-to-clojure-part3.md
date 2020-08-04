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
keys. Now that the trivial stuff are done, let's start writing the
property:

```clojure {linenos=table}
(deftest cache-specification-test
  (is (specification-correct? cache-specification)))
```

That's it: `stateful-check.core/specification-correct?` checks the
correctness of the given specification:

```clojure {linenos=table}
(def cache-specification
  {:commands          {:cache!  #'cache-command
                       :find    #'find-command
                       :flush!  #'flush-command}
   :generate-command  (fn [_state]
                        (gen/frequency [[1 (gen/return :cache!)]
                                        [3 (gen/return :find)]
                                        [1 (gen/return :flush!)]]))
   :initial-state     (constantly [])
   :setup             #(c/init! cache-size)})
```

The specification lays out the following:[^2]

* `:commands` specifies map of commands stateful-check can generate the
  sequence of actions
* `:generate-command` determines the frequency of the commands to generate
  (and in this case, the frequencies match the ones in the book)
* `:setup` performs one-time setup tasks to prepare the system-under-test
  (and in this case, it initializes the cache by calling `pbtic.cache/init!`)
* `initial-state` initializes the model/state that is used extensively
  during generation and execution

When `:setup` is given in the specification, the result of running the
setup will also be given to `initial-state`'s function; such behavior is
the reason why `initial-state` in the specification above is given as
`(constantly [])`, instead of just `vector`. If I were to do the latter,
as in the following:

```clojure {linenos=table}
(def another-specification
  {:commands      {:action #'action-command
                   :query  #'query-command}
   :initial-state vector
   :setup         (constantly {:hello :world})})
```

The initial state won't be an empty vector but a vector that contains
the hash-map, i.e., `[{:hello, :world}]`.[^3]

Such behavior is useful in certain scenarios, e.g., when the model/state
needs to know the state of the system-under-test in order to initialize
correctly.

Back to writing the property. I've chosen to use a vector as the model
to check against the cache: at anytime, the vector should hold up to 10
hash-maps, where each hash-map takes the form of `{:k k :v v}`. The easiest
command to implement is `flush-command`, which is shown below:

```clojure {linenos=table}
(def flush-command
  {:command       c/flush!
   :requires      (fn [state] (pos? (count state)))
   :args          (constantly nil)
   :precondition  (constantly true)
   :next-state    (constantly [])
   :postcondition (fn [_prev-state next-state _args _result]
                    (zero? (count next-state)))})
```

The snippet above shows all the entries stateful-check can work with:
they correspond closely to [PropEr][proper]/[propcheck][propcheck], except
stateful-check also allows specifying `:requires` as the precondition for
generating the command. Other than `:command`, the rest are optional.
In the snippet above, `:args` and `:precondition` are repeating the default
values that stateful-check uses, thus I could simplify the above as
follows:[^4]

```clojure {linenos=table}
(def flush-command
  {:command       c/flush!
   :requires      (fn [state] (pos? (count state)))
   :next-state    (constantly [])
   :postcondition (fn [_prev-state next-state _args _result]
                    (zero? (count next-state)))})
```

The [command specification][command-specs] at stateful-check's repo contains
more information. stateful-check expects interaction with the
system-under-test through `:command` and updates to the model/state through
`:next-state`. All others are predicates that steer the generation and
execution.

`flush-command` requires the model/state to be populated as the pre-requisite
for running: it makes no sense in running flush otherwise. While this
pre-requisite could also be enshrined by `:precondition`, checking it
earlier means skipping the argument generation step. After running,
`:next-state` updates the model/state by replacing it with an empty vector.
Strictly speaking, `:postcondition` isn't necessary for this; I added it
for instructional purposes.


[^1]: This expression creates a generator that generates positive numbers.
[^2]: You can read more about the specifications
      at https://github.com/czan/stateful-check/blob/master/doc/specification.org#system-specifications
[^3]: This bit me, leaving me bewildered for days, when I was learning stateful-check :joy:
[^4]: Not that it's a bad thing but the defaults sometimes left me wondering why
      my `:post-condition` isn't being run :facepalm:

[prev]: ../2020-07-10-property-based-testing-from-elixir-to-clojure-part2/
[pbtpee]: https://pragprog.com/book/fhproper/property-based-testing-with-proper-erlang-and-elixir
[ets]: http://erlang.org/doc/man/ets.html
[erlang-term]: http://erlang.org/doc/reference_manual/data_types.html#term
[stateful-check]: https://github.com/czan/stateful-check
[test.check]: https://github.com/clojure/test.check
[proper]: https://proper-testing.github.io/index.html
[propcheck]: https://github.com/alfert/propcheck
[command-specs]: https://github.com/czan/stateful-check/blob/master/doc/specification.org#commands
