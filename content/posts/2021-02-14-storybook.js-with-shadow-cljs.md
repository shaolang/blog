---
title: "Storybook.JS with Shadow-CLJS"
date: 2021-02-14T08:45:00+08:00
allowComments: true
tags: [clojure, tutorial]
---

[Storybook.JS][storybook.js] is a very interesting development tool from
JavaScript ecosystem[^1]. This tutorial shows how we can use it with
[Shadow-CLJS][shadow-cljs]. The code resides at
[storybook.js-with-shadow-cljs repo][repo].

## Prerequisites

The tutorial uses the following:

* [Java][java] version 11
* [Node.js][node] version 14.15.4
* [Reagent][reagent] version 1.0.0
* [Shadow-CLJS][shadow-cljs] version 2.11.8
* [Storybook.JS][storybook.js] version 6.1.17

Make sure the first two are installed prior the tutorial. The others will be
installed along the way.

## Getting a simple React app running

Let's create the scaffold to kick-start:

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

Line 8 adds [Reagent][reagent] as a dependency; lines 11 and 12 create the
profile `:frontend` (that matches the npm script's `shadow-cljs watch` command).
This profile specifies that the build targets the browser and should
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
an `h1` element and the `init` function that renders the header. To see this
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

By default, [Shadow-CLJS][shadow-cljs] generates the output to `public/js`,
hence the highlighted line (line 9). When the page is ready, `init` will run and
renders the header component. Before running `npm run dev`, add `dev-http`
to `shadow-cljs.edn` to configure the dev-server to listen to port 8080 and
serve artifacts from `public` directory:

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

With all these set up, run `npm run dev` and load
the page `localhost:8080` in your favorite browser; you should see "Hello,
World!":

![screenshot of "Hello, World!" rendered in browser](/images/2021-02-14-hello-world.png)

## Some cleanup

Before integrating with [Storybook.JS][storybook.js], let's do some cleaning
up: extract the custom `header` component to its own namespace and make
`acme.core/init` use that extracted one instead. First, the extracted
component at `src/main/acme/components/header.cljs`:

```clojure {linenos=table}
(ns acme.components.header)

(defn header [text]
  [:h1 text])
```

Then, in `src/main/acme/core.cljs`, delete `header` function and `require`
the header component namespace (as shown in line 2 below):

```clojure {linenos=table, hl_lines=[2]}
(ns acme.core
  (:require [acme.components.header :refer [header]]
            [reagent.dom :refer [render]]))

(defn init []
  (render [header "Hello, World!"]
          (js/document.getElementById "app")))
```

## Adding Storybook.JS to the mix

Time to add [Storybook.JS][storybook.js] to the project. Install it with
`npm install --save-dev @storybook/react`; then create `.storybook/main.js`
with the following contents to configure Storybook.JS to look for stories
in `public/js/stories` directory:

```javascript {linenos=table}
module.exports = {
  stories: ['../public/js/stories/**/*_stories.js'],
};
```

Update `shadow-cljs.edn` to create a new profile specifically for stories
that outputs the transpiled stories to `public/js/stories` too:

```clojure {linenos=table, hl_lines=[4,16,17,18]}
;; shadow-cljs configuration
{:source-paths
  ["src/dev"
   "src/main"
   "src/stories"
   "src/test"]

 :dev-http {8080 "public"}

 :dependencies
 [[reagent "1.0.0"]]

 :builds
 {:frontend {:target  :browser
             :modules {:main {:init-fn acme.core/init}}}
  :stories  {:target      :npm-module
             :entries     [acme.stories.header-stories]
             :output-dir  "public/js/stories"}}}
```

A few notable points on the new `:stories` profile:

* `:entries` specifies the namespaces to transpile to stories; unlike
  `:frontend` profile that specifies the target filename to output to
  (`main.js`), [Shadow-CLJS][shadow-cljs] uses the namespace as the output
  filename, e.g., `acme.stories.header_stories.js`
* `:target` states the build should target npm module which works for
  [Storybook.JS][storybook.js][^2]

Add two script commands to `package.json` to ease the auto-compilation of
stories and to start Storybook.JS:

```json {linenos=table, hl_lines=[3,4]}
"scripts": {
  "dev": "shadow-cljs watch frontend",
  "dev-stories": "shadow-cljs watch stories",
  "storybook": "start-storybook"
}
```

And finally, the story. Let' create a very simple story at
`src/stories/acme/stories/header_stories.cljs` that says "Hello, World!":

```clojure {linenos=table}
(ns acme.stories.header-stories
  (:require [acme.components.header :refer [header]]
            [reagent.core :as r]))

(def ^:export default
  #js {:title     "Header Component"
       :component (r/reactify-component header)})

(defn ^:export HelloWorldHeader []
  (r/as-element [header "Hello, World!"]))
```

The snippet above uses [Component Story Format][csf], hence the need to
add the metadata `^:export` to `default` and `HelloWorldHeader`. Because
[Storybook.JS][storybook.js] operates on React components, `reactify-component`
at line 7 turns the [Reagent][reagent] component into a React
one.[^3] With all these preparation, run `npm run dev-stories` in one console,
and `npm run storybook` in another. You should see [Storybook.JS][storybook.js]
render our first story:

![Rendering of Header component in Storybook](/images/2021-02-14-hello-world-in-storybook.png)

For the fun of it, let' append another story to `header-stories`:

```clojure {linenos=table}
(defn ^:export GoodbyeSekaiHeader []
  (r/as-element [header "Goodbye, Sekai!"]))
```

![Another rendering of Header component in Storybook](/images/2021-02-14-goodbye-sekai-in-storybook.png)

## Wrapping up

That concludes this tutorial on using [Storybook.JS][storybook.js] with
[Shadow-CLJS][shadow-cljs]. In this case, we are using [Reagent][reagent]
to create the components for [Storybook.JS][storybook.js] to render.
It shouldn't be that difficult to adapt the setup to work with other
ClojureScript rendering libraries, e.g., [Helix][helix].

[^1]: ClojureScript world also has a similar [devcards][devcards].
[^2]: Shadow-CLJS has a new `:esm` target that outputs to ES Modules,
      [but as of this writing][esm], it is cumbersome to use (the `^:export`
      metadata hint isn't working, thus requiring the need to declare all
      exports in `shadow-cljs.edn`.
[^3]: Refer to Reagent's tutorial on [Interop with React][reagent-iwr]
      for more information.

[storybook.js]: https://storybook.js.org
[shadow-cljs]: https://github.com/thheller/shadow-cljs
[repo]: https://github.com/shaolang/storybook.js-with-shadow-cljs
[node]: https://nodejs.org/en
[java]: https://jdk.java.net/java-se-ri/11
[devcards]: https://github.com/bhauman/devcards
[reagent]: https://reagent-project.github.io
[csf]: https://storybook.js.org/docs/react/writing-stories/introduction#component-story-format
[reagent-iwr]: https://cljdoc.org/d/reagent/reagent/1.0.0/doc/tutorials/interop-with-react
[helix]: https://github.com/lilactown/helix
[esm]: https://clojureverse.org/t/generating-es-modules-browser-deno/6116
