# Coercion Explained

Coercion is a process of transforming parameters (and responses) from one format into another. Reitit separates routing and coercion into two separate steps.

By default, all wildcard and catch-all parameters are parsed as Strings:

```clj
(require '[reitit.core :as r])

(def router
  (r/router
    ["/:company/users/:user-id" ::user-view]))
```

Match with the parsed `:params` as Strings:

```clj
(r/match-by-path r "/metosin/users/123")
; #Match{:template "/:company/users/:user-id",
;        :data {:name :user/user-view},
;        :result nil,
;        :params {:company "metosin", :user-id "123"},
;        :path "/metosin/users/123"}
```

To enable parameter coercion, the following things need to be done:

1. Define a `Coercion` for the routes
2. Define types for the parameters
3. Compile coercers for the types
4. Apply the coercion

## Define Coercion

`reitit.coercion/Coercion` is a protocol defining how types are defined, coerced and inventoried.

Reitit ships with the following coercion modules:

* `reitit.coercion.schema/coercion` for [plumatic schema](https://github.com/plumatic/schema)
* `reitit.coercion.spec/coercion` for both [clojure.spec](https://clojure.org/about/spec) and [data-specs](https://github.com/metosin/spec-tools#data-specs)

Coercion can be attached to route data under `:coercion` key. There can be multiple `Coercion` implementations within a single router, normal [scoping rules](../basics/route_data.html#nested-route-data) apply.

## Defining parameters

Route parameters can be defined via route data `:parameters`. It has keys for different type of parameters: `:query`, `:body`, `:form`, `:header` and `:path`. Syntax for the actual parameters depends on the `Coercion` implementation.

Example with Schema path-parameters:

```clj
(require '[reitit.coercion.schema])
(require '[schema.core :as s])

(def router
  (r/router
    ["/:company/users/:user-id" {:name ::user-view
                                 :coercion reitit.coercion.schema/coercion
                                 :parameters {:path {:company s/Str
                                                     :user-id s/Int}}}]))
```

A Match:

```clj
(r/match-by-path r "/metosin/users/123")
; #Match{:template "/:company/users/:user-id",
;        :data {:name :user/user-view,
;               :coercion <<:schema>>
;               :parameters {:path {:company java.lang.String,
;                                   :user-id Int}}},
;        :result nil,
;        :params {:company "metosin", :user-id "123"},
;        :path "/metosin/users/123"}
```

Coercion was not applied. Why? In Reitit, routing and coercion are separate processes and we haven't applied the coercion yet. We need to apply it ourselves after the successfull routing.

But now we should have enough data on the match to apply the coercion.

## Compiling coercers

Before the actual coercion, we need to compile the coercers against the route data. Compiled coercers yield much better performance and the manual step of adding a coercion compiler makes things explicit and non-magical.

Compiling can be done via a Middleware, Interceptor or a Router. We apply it now at router-level, effecting all routes (with `:parameters` and `:coercion` defined).

There is a helper function `reitit.coercion/compile-request-coercers` just for this:

```clj
(require '[reitit.coercion :as coercion])
(require '[reitit.coercion.schema])
(require '[schema.core :as s])

(def router
  (r/router
    ["/:company/users/:user-id" {:name ::user-view
                                 :coercion reitit.coercion.schema/coercion
                                 :parameters {:path {:company s/Str
                                                     :user-id s/Int}}}]
    {:compile coercion/compile-request-coercers}))
```

Routing again:

```clj
(r/match-by-path r "/metosin/users/123")
; #Match{:template "/:company/users/:user-id",
;        :data {:name :user/user-view,
;               :coercion <<:schema>>
;               :parameters {:path {:company java.lang.String,
;                                   :user-id Int}}},
;        :result {:path #object[reitit.coercion$request_coercer$]},
;        :params {:company "metosin", :user-id "123"},
;        :path "/metosin/users/123"}
```

The compiler added a `:result` key into the match (done just once, at router creation time), which holds the compiled coercers. We are almost done.

## Applying coercion

We can use a helper function `reitit.coercion/coerce!` to do the actual coercion, based on a `Match`:

```clj
(coercion/coerce!
  (r/match-by-path router "/metosin/users/123"))
; {:path {:company "metosin", :user-id 123}}
```

We get the coerced paremeters back. If a coercion fails, a typed (`:reitit.coercion/request-coercion`) ExceptionInfo is thrown, with data about the actual error:

```clj
(coercion/coerce!
  (r/match-by-path router "/metosin/users/ikitommi"))
; => ExceptionInfo Request coercion failed:
; #CoercionError{:schema {:company java.lang.String, :user-id Int, Any Any},
;                :errors {:user-id (not (integer? "ikitommi"))}}
; clojure.core/ex-info (core.clj:4739)
```

## Full example

Here's an full example for doing both routing and coercion with Reitit:

```clj
(require '[reitit.coercion.schema])
(require '[reitit.coercion :as coercion])
(require '[reitit.core :as r])
(require '[schema.core :as s])

(def router
  (r/router
    ["/:company/users/:user-id" {:name ::user-view
                                 :coercion reitit.coercion.schema/coercion
                                 :parameters {:path {:company s/Str
                                                     :user-id s/Int}}}]
    {:compile coercion/compile-request-coercers}))

(defn match-by-path-and-coerce! [path]
  (if-let [match (r/match-by-path router path)]
    (assoc match :parameters (coercion/coerce! match))))

(match-by-path-and-coerce! "/metosin/users/123")
; #Match{:template "/:company/users/:user-id",
;        :data {:name :user/user-view,
;               :coercion <<:schema>>
;               :parameters {:path {:company java.lang.String,
;                                   :user-id Int}}},
;        :result {:path #object[reitit.coercion$request_coercer$]},
;        :params {:company "metosin", :user-id "123"},
;        :parameters {:path {:company "metosin", :user-id 123}}
;        :path "/metosin/users/123"}

(match-by-path-and-coerce! "/metosin/users/ikitommi")
; => ExceptionInfo Request coercion failed...
```

## Ring Coercion

For a full-blown http-coercion, see the [ring coercion](../ring/coercion.md).

## Thanks to

Most of the thing are just polished version of the original implementations. Thanks to:

* [compojure-api](https://clojars.org/metosin/compojure-api) for the initial `Coercion` protocol
* [ring-swagger](https://github.com/metosin/ring-swagger#more-complete-example) for the `:parameters` and `:responses` syntax.
* [schema](https://github.com/plumatic/schema) and [schema-tools](https://github.com/metosin/schema-tools) for Schema Coercion
* [spec-tools](https://github.com/metosin/spec-tools) for Spec Coercion