---
title: "Property-Based Testing: From Erlang/Elixir to Clojure"
date: 2019-08-12T00:00:00+08:00
allowComments: true
---

Reading [Property-Based Testing with PropEr, Erlang, and Elixir][pbtpee] and
following along the examples helped me in learning this exciting testing
methodology; but it also left me wondering: have I really absorbed and
internalized _just by following along_?

So, I reached out to [Fred][fred], got his approval, and started translating
the code from Erlang/Elixir to Clojure with [test.check][test.check].
All the code done [so far] are hosted
at <https://github.com/shaolang/pbtic> I'm using [test.check][test.check]
as the property-based testing tool in Clojure.

## Birthday Greeting Kata

The book breaks up the kata into 4 parts: [CSV parsing](#csv-parsing),
[records filtering](#records-filtering),
[employee module](#employee-module) (bridging CSV parsing and records filtering),
and [email templating](#email-templating).

### CSV Parsing

{{< highlight clojure "linenos=table" >}}
(ns pbtic.birthday.csv-test
  (:require [clojure.test :refer [deftest is]]
            [clojure.test.check.clojure-test :refer [defspec]]
            [clojure.test.check.generators :as gen]
            [clojure.test.check.properties :refer [for-all]]
            [pbtic.birthday.csv :as csv]))

;;;;;;;
;; defs

(def ^:private text-data
  (str "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789"
       ":;<=>?@ !#$%&'()*+-./[\\]^_`{|}~"))

;;;;;;;;;;;;;
;; generators

(defn- text [cs]
  (gen/let [xs (gen/list (gen/elements cs))]
    (apply str xs)))

(def unquoted-text (text text-data))
(def quotable-text (text (str text-data "\r\n\",")))


(def field (gen/one-of [unquoted-text, quotable-text]))


(def header (partial gen/vector field))
(def record (partial gen/vector field))


(defn entry [size ks]
  (gen/let [vs  (record size)]
    (zipmap ks vs)))


(def csv-source
  (gen/let [size    gen/pos-int
            ks      (header (inc size))]
    (gen/list (entry (inc size) ks))))

{{</ highlight >}}

Unlike `?LET`/`let` in [PropEr][proper]/[PropCheck][propcheck],
[`clojure.test.check.generators/let`][test.check/let] can have
multiple bindings where the binding after can reference the
value of the one before, as shown at lines 39 and 40.

`csv-source` generates a list of maps, where each map is one record from the
CSV source, and the keys of each map the header record. While it's idiomatic
for Clojure maps to use keywords as keys to maps, the book recommends keeping
the CSV parsing focused on encoding/decoding and not bring in any business
requirements to keep the module/namespace maintainable.

{{< highlight clojure "linenos=table,linenostart=43" >}}
;;;;;;;;;;;;;
;; properties

(defspec roundtrip-encoding-decoding
  (for-all [maps  csv-source]
    (= maps (csv/decode (csv/encode maps)))))

;;;;;;;;
;; tests

(deftest one-column-csv-files-are-inherently-ambiguous
  (is (= "\r\n\r\n\r\n"
         (csv/encode [{"" ""}, {"" ""}])))

  (is (= [{"" ""}]
         (csv/decode "\r\n\r\n"))))


(deftest one-record-per-line
  (is (= [{"aaa" "zzz", "bbb" "yyy", "ccc" "xxx"}]
         (csv/decode "aaa,bbb,ccc\r\nzzz,yyy,xxx\r\n"))))


(deftest optional-trailing-crlf
  (is (= [{"aaa" "zzz", "bbb" "yyy", "ccc" "xxx"}]
         (csv/decode "aaa,bbb,ccc\r\nzzz,yyy,xxx"))))


(deftest double-quotes
  (is (= [{"aaa" "zzz", "bbb" "yyy", "ccc" "xxx"}]
         (csv/decode "\"aaa\",\"bbb\",\"ccc\"\r\n\"zzz\",\"yyy\",\"xxx\""))))


(deftest escape-crlf
  (is (= [{"aaa" "zzz", "b\r\nbb" "yyy", "ccc" "xxx"}]
         (csv/decode "\"aaa\",\"b\r\nbb\",\"ccc\"\r\nzzz,yyy,xxx"))))


(deftest double-quotes-escaping
  (is (= [{"aaa" "", "b\"bb" "", "ccc" ""}]
         (csv/decode "\"aaa\",\"b\"\"bb\",\"ccc\"\r\n,,"))))


(deftest dupe-keys-unsupported
  (let [csv     (str "field_name,field_name,field_name\r\n"
                     "aaa,bbb,ccc\r\n"
                     "zzz,yyy,xxx\r\n")
        [m1 m2] (csv/decode csv)]
    (is (= ["field_name"] (keys m1)))
    (is (= ["field_name"] (keys m2)))))
{{</ highlight >}}

The property and tests straight-up mirror the original code in the book.
But the implementation? LOL, I took the shortcut by using [data.csv][data.csv]
instead, as my main intention is to learn how to write property tests, not
implementing a CSV parser.

{{< highlight clojure "linenos=table" >}}
(ns pbtic.birthday.csv
  (:require [clojure.data.csv :as csv]
            [clojure.string :as str]))

(defn encode [ms]
  (let [ks    (-> ms first keys)
        vs    (map vals ms)
        out   (java.io.StringWriter.)]
    (csv/write-csv out (conj vs ks) :newline :cr+lf)
    (.toString out)))


(defn decode [s]
  (let [[header & body] (csv/read-csv (java.io.StringReader. s))]
    (map (partial zipmap header) body)))
{{</ highlight >}}

## Records Filtering

Records filtering module/namespace filters the employees whose birthdays fall
on the given date. Although properties are the new-found power, employing them
for this instance might not be suitable, as property tests are probabilistic
at their core. "Traditional" unit tests can explore this problem space much
better. Even so, Fred suggests running an exhaustive search to cover all
possible cases, using generators in property-based tests as inspiration in
"generating" the cases.

{{< highlight clojure "linenos=table" >}}
(ns pbtic.birthday.bday-filter-test
  (:require [pbtic.birthday.bday-filter :as filter :refer [month-day]]
            [clojure.set :as set]
            [clojure.test :refer [deftest is]])
  (:import [java.time DateTimeException LocalDate]))

;;;;;;;;;;
;; helpers

(defn find-birthdays-for-year [people yeardata]
  (when (seq yeardata)
    (let [[day & year]  yeardata
          found         (filter/birthday people day)]   ;; <- function being tested
      (assoc (find-birthdays-for-year people year) day found))))


(defn generate-year-data [start]
  (let [start-date  (LocalDate/of start 1 1)
        end-date    (LocalDate/of (inc start) 1 1)]
    (into [] (.. start-date (datesUntil end-date) toArray))))


(defn generate-years-data [start end]
  (mapv generate-year-data (range start (inc end))))


(defn rand-name []
  (apply str (repeatedly 30 #(rand-nth "abcdefghijklmnopqrstuvwxyz"))))


(defn people-for-date [date]
  (try
    (let [[month day] (month-day date)
          rand-year   (+ 1900 (rand-int 100))]
      {:name          (rand-name)
       :date-of-birth (LocalDate/of rand-year month day)})
    (catch Exception _ (people-for-date date))))


(defn people-for-year [year]
  (map people-for-date year))


(defn generate-people-for-year [n]
  (let [year-seed (generate-year-data 2016)]  ;; leap year so all days are covered
    (mapcat (fn [_] (people-for-year year-seed)) (range n))))

;;;;;;;;;;;;;
;; assertions

(defn every-birthday-once [people birthdays]
  (let [found       (mapcat second birthdays)
        not-found   (set/difference (set people) (set found))]
    (is (empty? not-found))
    (is (zero? (- (count found) (count (set found)))))))


(defn on-right-date [people birthdays]
  (doseq [[date found]            birthdays
          {:keys [date-of-birth]} found]
    (let [[dob-month dob-day] (month-day date-of-birth)]
      (try
        (LocalDate/of (.getYear date) dob-month dob-day)
        (is (= (month-day date)
               (month-day date-of-birth)))
        (catch DateTimeException _ true)))))

;;;;;;;
;; test

(deftest property-style-filtering
  (let [years   (generate-years-data 2018 2038)
        people  (generate-people-for-year 3)]
    (doseq [yeardata years]
      (let [birthdays (find-birthdays-for-year people yeardata)]
        (every-birthday-once people birthdays)
        (on-right-date people birthdays)))))
{{</ highlight >}}

The differences between this and the original are:

* `generate-year-data` (lines 17-20) is significantly shorter than the original,
  as it uses [`java.time.LocalDate#datesUntil`][datesUntil] method to generate
  the dates. The Elixir version in the book is long probably because Fred
  wanted to align it with the Erlang version; if he were to use Elixir's
  [Date.range/2][elixir-date-range] function (no equivalent available in
  Erlang :sob:), the code would be much shorter, as shown below:

{{< highlight elixir "linenos=table" >}}
defp generate_year_data(year) do
  {:ok, start_date} = Date.new(year, 1, 1)
  {:ok, end_date} = Date.new(year, 12, 31)

  Date.range(start_date, end_date)
  |> Enum.into([])
end
{{</ highlight >}}

* `rand-name` (lines 27-28) is a simple stand-in for Erlang's `make-ref/0`.
* `every-birthday-once` (line 51-55) uses the set data structure to determine
  the set of people not found.
* `on-right-date` (lines 58-66) catches invalid dates and returns `true` in
  the catch clause to signify "skipping," as Clojure does not use
  pattern-matching as much as Erlang/Elixir do.

The implementation of `pbtic.birthday.bday-filter` is as follows:

{{< highlight clojure "linenos=table" >}}
(ns pbtic.birthday.bday-filter)


(def month-day (juxt #(.getMonthValue %) #(.getDayOfMonth %)))


(defn birthday-no-leap-year-handling [people date]
  (let [md (month-day date)]
    (filter #(= (month-day (:date-of-birth %)) md) people)))


(defn  filter-dob [people month day]
  (filter #(= (month-day (:date-of-birth %)) [month day]) people))


(defn birthday [people date]
  (let [[month day] (month-day date)]
    (if (and (= [month day] [2 28]) (not (.isLeapYear date)))
      (concat (filter-dob people 2 28) (filter-dob people 2 29))
      (filter-dob people month day))))
{{</ highlight >}}

`birthday-no-leap-year-handling` (lines 7-9) shows how the code looks
like prior to handling of 29 Feb birthdays on non-leap year (the company
shouldn't only wish them happy birthday once every four years, should they?).

### Employee Module

The namespace to rule them all :ring:; it brings `pbtic.birthday.csv` and
`pbtic.birthday.bday-filter` together. Functions in this namespace
implement business requirements that other namespace may not have already,
e.g., the CSV parser in `pbtic.birthday.csv`.

{{< highlight clojure "linenos=table" >}}
(ns pbtic.birthday.employee-test
  (:require [clojure.string :as str]
            [clojure.test.check.clojure-test :refer [defspec]]
            [clojure.test.check.generators :as gen]
            [clojure.test.check.properties :refer [for-all]]
            [pbtic.birthday.csv-test :as csv-test]
            [pbtic.birthday.employee :as employee])
  (:import [java.time LocalDate]))

;;;;;;;
;; defs

(def start-date (LocalDate/of 1900 1 1))

(def max-days (.. start-date
                  (until (LocalDate/of 2021 1 1))
                  getDays))

;;;;;;;;;;;;;
;; generators

(def text-date
  (gen/let [days-to-add (gen/choose 0 max-days)]
    (let [date (.plusDays start-date days-to-add)]
      (format " %4d/%02d/%02d"
              (.getYear date)
              (.getMonthValue date)
              (.getDayOfMonth date)))))


(def whitespaced-text
  (gen/let [txt csv-test/field]
    (str " " txt)))


(def raw-employee-map
  (gen/let [val-list (gen/tuple csv-test/field
                                whitespaced-text
                                text-date
                                whitespaced-text)]
    (zipmap ["last_name", " first_name", " date_of_birth", " email"] val-list)))

;;;;;;;;;;;;;
;; properties

(defspec check-that-leading-space-is-fixed
  (for-all [m raw-employee-map]
    (let [emp (employee/adapt-csv-result m)]
      (every? #(not (str/starts-with? (name %) " "))
              (concat (keys emp)
                      (filter string? (vals emp)))))))


(defspec check-that-date-is-formatted-right
  (for-all [m raw-employee-map]
    (let [m (employee/adapt-csv-result m)]
      (= (type (get m :date-of-birth)) LocalDate))))
{{</ highlight >}}

Taking Fred's advice on reworking a restriction into a transformation when
creating custom generators (refer to Imposing Restrictions subsection under
Basic Custom Generators in chapter 4 of the book), `text-date` (lines 22-28)
ditches `gen/such-that` and goes for a transformation that adds days to a
known start date (1900-01-01). Although this is not a pure code-porting, it
shows how re-imagining the problem on-hand is not that all difficult.

{{< highlight clojure "linenos=table,hl_lines=19" >}}
(ns pbtic.birthday.employee
  (:require [clojure.string :as str]
            [pbtic.birthday.bday-filter :as bday-filter]
            [pbtic.birthday.csv :as csv])
  (:import [java.time LocalDate]
           [java.time.format DateTimeFormatter]))

;;;;;;;
;; defs

(def ^:private datetime-formatter (DateTimeFormatter/ofPattern "yyyy/MM/dd"))

;;;;;;;;;;;;;
;; public API

(defn adapt-csv-result [m]
  (let [ks  (sequence (comp (map str/triml)
                            (map #(str/replace % #"_" "-"))
                            (map keyword))
                      (keys m))
        m   (zipmap ks (map str/triml (vals m)))
        dob (:date-of-birth m)]
    (assoc m :date-of-birth (LocalDate/parse dob datetime-formatter))))


(defn from-csv [s]
  (map adapt-csv-result (csv/decode s)))


;; parameter order is different from the book's
(defn filter-birthday [date employees]
  (bday-filter/birthday employees date))
{{</ highlight >}}

`pbtic.birthday.employee` namespace does not implement "accessors" because
of the keyword-izing of keys at line 19 because of Clojure's idiom on
using keywords to retrieve values from maps (line 22 shows one such example).
The other change is the swapping of the parameters in `filter-birthday` (lines
31-32); this makes the function more easily usable with the [`->>`][->>]
threading macro; Elixir [`|>`][|>] expects collections to be the first argument,
but Clojure `->>` expects it to be the last). Such omission makes Clojure's
implementation/port shorter.

### Templating

`pbtic.birthday.mail-tpl/body` function creates the message for
inserting into the email. It's relatively easy that could have been
tested using traditional unit testing method...

{{< highlight clojure "linenos=table" >}}
(ns pbtic.birthday.mail-tpl-test
  (:require [clojure.string :as str]
            [clojure.test.check.clojure-test :refer [defspec]]
            [clojure.test.check.generators :as gen]
            [clojure.test.check.properties :refer [for-all]]
            [pbtic.birthday.csv-test :as csv-test]
            [pbtic.birthday.mail-tpl :as mail-tpl])
  (:import [java.time LocalDate]))

;;;;;;;;;;;;;
;; generators

(def date
  (gen/let [days-to-add gen/nat]
   (.plusDays (LocalDate/of 1900 1 1) days-to-add)))


(def employee-map
  (gen/let [vs (gen/tuple (gen/not-empty csv-test/field)
                          (gen/not-empty csv-test/field)
                          date
                          (gen/not-empty csv-test/field))]
    (zipmap [:last-name :first-name :date-of-birth :email] vs)))

;;;;;;;;;;;;;
;; properties

(defspec email-template-has-first-name
  (for-all [employee employee-map]
    (str/includes? (mail-tpl/body employee)
                   (:first-name employee))))
{{</ highlight >}}

The implementation is trivial:

{{< highlight clojure "linenos=table" >}}
(ns pbtic.birthday.mail-tpl)

(defn body [{:keys [first-name]}]
  (format "Happy birthday, dear %s!" first-name))


(defn full [{:keys [email] :as employee}]
  [email, "Happy birthday!", (body employee)])
{{</ highlight >}}

### Plumbing It All Together

`pbtic.birthday/run` function ties all these up (without any integration tests
written).

{{< highlight clojure "linenos=table" >}}
(ns pbtic.birthday
  (:require [pbtic.birthday.csv :as csv]
            [pbtic.birthday.employee :as employee]
            [pbtic.birthday.mail-tpl :as mail-tpl])
  (:import [java.time LocalDate]))

;;;;;;;;;
;; helper

(defn- send-email [[to, _topic, _body]]
  (println "sent birthday email to" to))

;;;;;;;;;;;;;
;; public api

(defn run [path & {:keys [curr-date] :or {curr-date (LocalDate/now)}}]
  (doseq [employee (->> (slurp path)
                        employee/from-csv
                        (employee/filter-birthday curr-date))]
    (send-email (mail-tpl/full employee))))
{{</ highlight >}}

To test it, pass the file name of the employee record CSV and
optionally the "current date" (defaults to today). For example, run the
following at Clojure REPL:

```clojure-repl
user=> (require '[pbtic.birthday :as bday])
user=> (import '[java.time LocalDate])
user=> (bday/run "resources/birthday/employees.csv" :curr-date (LocalDate/of 2019 8 10))
sent birthday email to john.doe@foobar.com
nil
```

## Summary

It's quite a blast in porting the code from Erlang/Elixir to Clojure. In the
[next post][part2], I'll cover the code ported from chapter 6
"Properties-Driven Development."

[pbtpee]: https://pragprog.com/book/fhproper/property-based-testing-with-proper-erlang-and-elixir
[fred]: https://twitter.com/mononcqc
[test.check]: https://github.com/clojure/test.check
[proper]: https://github.com/proper-testing/proper
[propcheck]: https://github.com/alfert/propcheck
[test.check/let]: http://clojure.github.io/test.check/clojure.test.check.generators.html#var-let
[data.csv]: https://github.com/clojure/data.csv
[birthday.csv]: https://github.com/shaolang/pbtic/blob/master/src/pbtic/birthday/csv.clj
[datesUntil]: https://docs.oracle.com/javase/9/docs/api/java/time/LocalDate.html#datesUntil-java.time.LocalDate-
[elixir-date-range]: https://hexdocs.pm/elixir/Date.html#range/2
[->>]: https://clojure.github.io/clojure/clojure.core-api.html#clojure.core/-%3E%3E
[|>]: https://hexdocs.pm/elixir/Kernel.html#%7C%3E/2
[part2]: ../2020-07-10-property-based-testing-from-elixir-to-clojure-part2/
