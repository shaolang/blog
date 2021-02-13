---
title: "Tutorial: Storybook.JS with Shadow-CLJS"
date: 2021-02-13T11:39:43+08:00
draft: true
allowComments: true
tags: [clojure, tutorial]
---

[Storybook.JS][storybook.js] is a very interesting development tool from
JavaScript ecosystem[^1]. This tutorial shows how we can use it with
[Shadow-CLJS][shadow-cljs].

## Prerequisites

The tutorial uses the following:

* [Node.js][node] version 14.15.4
* [Java][java] version 11
* [Shadow-CLJS][shadow-cljs] version 2.11.8

The rest of the dependencies are stated in the tutorial.

## Getting a simple React app running

To ensure the setup is working, let's create the scaffold:

```bash
$ mkdir acme
$ cd acme
$ npm init        # just keep pressing enter until the prompt ends
$ npm install --save-dev shadow-cljs
```

In the generated `package.json`, add a helper script to launch shadow-cljs
and automatically compile when it detect changes:

```json
"scripts": {
  "dev": "shadow-cljs watch frontend"
}
```

The script uses the `:frontend` profile defined in `shadow-clj.edn` for
ClojureScript compiler. Run `npx shadow-cljs init` to generate the skeleton
`shadow-cljs.edn` file and edit it as follows:

```clojure {linenos=table, hl_lines=[8,11,12]}
;; shadow-cljs configuration
{:source-paths
  ["src/dev"
   "src/main"
   "src/test"]

 :dependencies
 [[reagent "1.0.0"]]

 :builds
 {:frontend {:target  :browser
             :modules {:main {:init-fn acme.core/init}}}}}
```

Line 8 adds [reagent][reagent] as a dependency; lines 11 and 12 creates the
profile `:frontend` (that matches the npm script's `shadowcljs watch` command).
In this profile, it specifies the build targets the browser and it should
generate the file `main.js` ('cos of the `:main` key) that will invoke
`acme.core/init` function at initialization. Let's implement `init` that uses
a simple Reagent component in `src/main/acme/core.cljs`:

```clojure {linenos=table}
(ns acme.core
  (:require [reagent.dom :refer [render]]))

(defn header [text]
  [:h1 text])

(defn init []
  (render [header "Hello, World!"]
          (js/document.getElementById "app")))
```

Simple enough: a custom `header` component that outputs the given text in
a `h1` element and the `init` function that renders the header. To see this
glorious app render, create the `public/index.html` as follows:

```html {linenos=table, hl_lines=[9]}
<!doctype html>
<html>
  <head>
    <meta charset='utf-8'>
    <title>Acme</title>
  </head>
  <body>
    <div id='app'></div>
    <script src='js/main.js'></script>
  </body>
</html>
```

By default, Shadow-CLJS generates the output to `public/js`, hence the
highlighted line (line 9). When the page is ready, `init` will run and
renders the header. Before running `npm run dev`, add `dev-http` config
to `shadow-cljs.edn` to set up a dev-server:

```clojure {linenos=table, hl_lines=[7]}
;; shadow-cljs configuration
{:source-paths
  ["src/dev"
   "src/main"
   "src/test"]

 :dev-http {8080 "public"}

 :dependencies
 [[reagent "1.0.0"]]

 :builds
 {:frontend {:target  :browser
             :modules {:main {:init-fn acme.core/init}}}}}
```

Line 7 configures a dev-server that listens at port 8080 and serves artifacts
in `public` directory. With all these set up, run `npm run dev` and load
the page `localhost:8080` in your favorite browser; you should see "Hello,
World!":

![screenshot of "Hello, World!" rendered in browser](/images/2021-02-14-hello-world.png)


[^1]: ClojureScript world also have the equivalent [devcards][devcards].

[storybook.js]: https://storybook.js.org
[shadow-cljs]: https://github.com/thheller/shadow-cljs
[node]: https://nodejs.org/en
[java]: https://jdk.java.net/java-se-ri/11
[devcards]: https://github.com/bhauman/devcards
[reagent]: https://reagent-project.github.io
