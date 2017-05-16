---
layout: post
title: "Using the Lacinia GraphQL Clojure Library"
description: ""
comments: true
category: 
tags: [lacinia, GraphQL]
---
{% include JB/setup %}

# Introduction

[GraphQL](http://graphql.org/) is a query language for Web APIs. It is an alternative to other API architectures such as REST. The advantages that GraphQL has over the traditional REST approach is that clients can have a more granular specification for the data they wish to access while also providing built-in documentation and type handling. The disadvantages are that most developers are familiar with the RESTful approach. However, GraphQL is gaining mindshare as Facebook, GitHub and Walmart all provide this type of API interface.

# Overview

GraphQL has two components

* The schema specification
* A client query language

The schema is analogous to db schemas in SQL. The query is analogous to operations in SQL. The format of GraphQL queries are defined in the schema.

The schema and the query language are written using javascript strings. The API developer defines a schema and the clients access it with the query language. All communication between server and client is in JSON.

# GraphQL Schema and Queries

[Lacinia](https://github.com/walmartlabs/lacinia) is an implementation of GraphQL written in Clojure. Instead of using javascript, the schema is defined using extensible data notation ([edn](https://github.com/edn-format/edn)). Here is an example of a schema that defines a single object called Game and a query to retrieve a game by its name:


```clojure
{:objects
  {:Game {:description "A game owned by a developer for which scores can be recorded"
          :fields {:key {:type (non-null String)
                         :description "Unique identifier for this game"}
                   :name {:type (non-null String)}
                   :created {:type Int
                             :description "Unix epoch seconds when game was added to database"}}}}
 :queries
  {:game {:type :Game
          :description "Retrieve a single Game by its name"
          :args {:name {:type (non-null String)
                        :description "Unique name for game."}}
          :resolve :resolve-game}}}
```

The Game object is composed of its description and fields keys. The description is a string which describes the object. The fields are types and can be:

* A keyword corresponding to an object, interface, enum, or union
* A scalar type (built in, or schema defined)
* A non-nillable version of any of the above: (non-null X)
* A list of any of the above: (list X)

The built-in scalar types are:

* String
* Float
* Int
* Boolean
* ID

(from [Lacinia docs](http://lacinia.readthedocs.io/en/latest/fields.html))

Each field can have its own description. GraphQL APIs are self-documenting in this respect. Instead of providing API documentation in a separate HTML document, the GraphQL URL for a service is given to a client for them to explore with the [GraphiQL](https://github.com/graphql/graphiql) tool. This tool can be self-hosted by the API service or run on the client's machine.

Documentation for the Game object would appear in the GraphiQL tool like this:

![Game documentation](/assets/images/graphql-game-documentation.png)

The 'created' field provides more information. Its documentation looks like this in GraphiQL

![created field documentation](/assets/images/graphql-game-created-documentation.png)

A Game object can be accessed via the following query:

```javascript
{
  game(name: "Greedy Pigeon") {
    key
    created
  }
}
```

Which results in the following response:

```javascript
{
 "data": {
   "game": {
     "key": "492ddc12cae34b898cfeee4727ac8b96",
     "created": 1494383617
    }
  }
}
```


In GraphiQL, this looks like:

![graphql example query](/assets/images/graphql-example-query.png)

A great way to begin exploring GraphQL is to have a look at Github's [online GraphQL API](https://developer.github.com/early-access/graphql/explorer/). For a more in-depth understanding, access Github's GraphiQL API [as a developer](https://developer.github.com/early-access/graphql/guides/accessing-graphql/).

# Processing the Lacinia edn schema

The schema is defined in an edn file. The data for the fields is retrieved using a resolver. As edn is data, not code, resolvers must be attached to the edn schema with the **attach-resolvers** function. Finally, the schema is compiled with the **schema/compile** function.

```clojure
(ns leaderboard-api.schema
 (:require [leaderboard-api.db :as db]
           [clojure.java.io :as io]
           [clojure.edn :as edn]
           [com.walmartlabs.lacinia.schema :as schema]
           [com.walmartlabs.lacinia.util :refer [attach-resolvers]]))

(defn leaderboard-schema
 []
 (-> (io/resource "edn/leaderboard-schema.edn")
     slurp
     edn/read-string
     (attach-resolvers {:resolve-game db/resolve-game})
     schema/compile)
```

The example **leaderboard-schema.edn** is located in *leaderboard-api/resources/edn/leaderboard-schema.edn*. The **db/resolve-game** function is a database call to retrieve a game object from the database. It is defined in the leaderboard-api.db namespace.

```clojure
(ns leaderboard-api.db
  (:require [clojure.data.json :as json]
            [environ.core :refer [env]]
            [leaderboard-api.core :as core]
            [yesql.core :refer [defqueries]]))

;; still need to put a password in for this
;; need to be sure the database is password protected!
(def db-spec {:classname "org.postgresql.Driver"
              :subprotocol "postgresql"
              :subname (str "//"
                            (or (:db-host env)
                                (System/getenv "OPENSHIFT_PG_HOST"))
                            ":"
                            (or (:db-port env)
                                (System/getenv "OPENSHIFT_PG_PORT"))
                            "/"
                            (or (:db-name env)
                                (System/getenv "OPENSHIFT_PG_DATABASE")))
              :user (or (:db-username env)
                        (System/getenv "OPENSHIFT_PG_USERNAME"))
              :password (or (:db-password env)
                            (System/getenv "OPENSHIFT_PG_PASSWORD"))})

(defqueries "sql/operations.sql"
  {:connection db-spec})

;; see: https://gist.github.com/alexpw/2166820
(defmacro check-error
  "Usage: (check-error (create-developer! (core/new-developer \"foo@bar.com\")))"
  [body]
  `(try ~body (catch Exception e# (throw (Exception.(:cause (Throwable->map (.getNextException e#))))))))

(defn resolve-game
  [context args _value]
  (let [developer (:authorization @(:cache context))]
    (first
     (check-error (get-game (assoc args :developer developer))))))
     
```
The **get-game** function is a call to the PostgreSQL database. **check-error** is a macro used to catch any exceptions thrown by the PostgreSQL database, retrieve the error message and then throw it wrapped in an exception (this rounadabout way of doing things is due to the fact that the PostgreSQL driver throws an exception with a generic error message that indicates what further action must be done in order to get the actual error message that gave rise to the original exception!). This exception will be handled by Lacinia and returned as part of the "error" key in the JSON response.

Lacinia will pass three arguments to the data resolver. **context** is an atom which contains a context map (The **:authorization** key was added by the ring server handler and is discussed further below). The **args** argument is a map of arguments defined in the GraphQL query. The example query above would pass a map like {:name "Greedy Pigeon"} as the arguments map. The **_value** argument is not needed in this context, but corresponds to the fields's resolved value. Further description is beyond the scope of this article, but more can be read about it [here](http://lacinia.readthedocs.io/en/latest/resolve/overview.html#container-s-resolved-value).

The get-game Clojure function has been transformed from SQL by the [Yesql](https://github.com/krisajenkins/yesql) **defqueries** macro. It is easily surmised from the following SQL that **get-game** is a function that accepts a map of the form {:name :developer } and returns a game object from the database.

```sql
-- name: get-game
SELECT key,name,created FROM games WHERE name = :name AND developer = :developer;
```
# The *ring* handler

Requests to the GraphQL query must be handled by a web server which can provide a JSON response. A good starting point can be found [here](https://github.com/hlship/boardgamegeek-graphql-proxy/blob/master/src/bgg_graphql_proxy/server.clj). The approach used here is to have the minimal setup required in order to serve GraphQL requests.

Here is the main web app handler through which requests are passed:

```clojure
  (-> handler
      wrap-params
      (wrap-cors :access-control-allow-origin [#".*"]
                 :access-control-allow-methods [:get :post])
      wrap-body-string))
```

The handler function function looks like this:

```clojure
(defn handler [request]
  (let [uri (:uri request)]
    (if (= uri "/graphql")
      ;; hits the proper uri, process request
      ((graphql-handler (leaderboard-schema)) request)
      ;; not serving any other requests
      {:status 404
       :headers {"Content-Type" "text/html"}
       :body (str "Only GraphQL JSON requests to /graphql are accepted on this server")})))
```
The **handler** fn filters out all requests, except those made to the /graphql URI.

The **wrap-cors** middleware responds to a browser's OPTIONS request and returns a simple ACCESS CONTROL ALLOW ORIGIN header which allows requests to the server from any domain. This enables a web game to access the Leaderboard API service through XHR requests. The security implications of this are worth discussing, but are outside of the scope of this article.

The **wrap-body-string** middleware looks like this:

```clojure
(defn wrap-body-string [handler]
  (fn [request]
    (let [body-str (body-string request)]
      (handler (assoc request :body body-str)))))
```

This is used to deal with the fact that the body of the request is a steam and is consumed when read. **wrap-body-string** transforms the request into an immutable data object by replacing the request body stream with an immutable string.

With some of the details of handling requests out of the way, the real work is left to the **graphql-handler** fn.

```clojure
(defn ^:private graphql-handler
  "Accepts a GraphQL query via GET or POST, and executes the query.
  Returns the result as text/json."
  [compiled-schema]
  (let [context {:cache (atom {})}]
    (fn [request]
      ;; include authorization key in context
      (swap! (:cache context) assoc :authorization
             (extract-authorization-key request))
      (let [vars (variable-map request)
            query (extract-query request)
            result (execute compiled-schema query vars context)
            status (if (-> result :errors seq)
                     400
                     200)]
        {:status status
         :headers {"Content-Type" "application/json"}
         :body (json/write-str result)}))))
```

The vars and query are extracted from the request, the result is computed by the lacinia **execute** fn using the schema and returned as a JSON response. The code for **extract-map** looks like this:

```clojure
(defn variable-map
  "Reads the `variables` query parameter, which contains a JSON string
  for any and all GraphQL variables to be associated with this request.
  Returns a map of the variables (using keyword keys)."
  [request]
  (let [variables (condp = (:request-method request)
                    ;; We do a little bit more error handling here in the case
                    ;; where the client gives us non-valid JSON. We still haven't
                    ;; handed over the values of the request object to lacinia
                    ;; GraphQL so we are still responsible for minimal error
                    ;; handling
                    :get (try (-> request
                                  (get-in [:query-params "variables"])
                                  (json/read-str :key-fn keyword))
                              (catch Exception e nil))
                    :post (try (-> request
                                   :body
                                   (json/read-str :key-fn keyword)
                                   :variables)
                               (catch Exception e nil)))]
    (if-not (empty? variables)
      variables
      {})))
```

There is additional error handling here because the variable map must be extracted from the JSON in the request. **variable-map** handles the case where invalid JSON is sent to the server.

The query is obtained, with similar error handling, by **extract-query** 

```clojure
(defn extract-query
  "Reads the `query` query parameters, which contains a JSON string
  for the GraphQL query associated with this request. Returns a
  string.  Note that this differs from the PersistentArrayMap returned
  by variable-map. e.g. The variable map is a hashmap whereas the
  query is still a plain string."
  [request]
  (case (:request-method request)
    :get  (get-in request [:query-params "query"])
    ;; Additional error handling because the clojure ring server still
    ;; hasn't handed over the values of the request to lacinia GraphQL
    :post (try (-> request
                   :body
                   (json/read-str :key-fn keyword)
                   :query)
               (catch Exception e ""))
    :else ""))
```

In addition to extracting the query and variable map, there is a step which extracts the developer key from an authorization header. Not all requests require the developer key (such as posting a score or obtaining high scores), but for those that do, it is obtained with **extract-authorization-key** and added to the context of a request in **graphql-handler**.

```clojure
(defn extract-authorization-key
  "Extract the authorization key from the request header. The
  authorization header is of the form: Authorization: bearer <key>"
  [request]
  (if-let [auth-header (-> request
                           :headers
                           (get "authorization"))]
    (-> auth-header
        (string/split #"\s")
        last)
    nil))
```

# Final thoughts

GraphQL allows for a richer, yet more granular Web API. The client can obtain only the information needed. Using a declarative approach makes API development simpler, while also being easier to maintain and reason about. The self-documenting nature of GraphQL eliminates the additional burden of documenting an API. In many respects, this is similar to the way documentation for a Clojure library is inferred from doc-strings. Design shifts away from the hard part of naming things (API routes) to providing data and documentation to the client.

In combination with a library such as Yesql, an API can be designed side-by-side with the underlying SQL calls using minimal wrapper code. This approach drastically simplifies Web API development.  The excellent Clojure GraphQL library Lacinia makes Web API development a joy.

Lacinia documentation can be found [here](http://lacinia.readthedocs.io/en/latest/overview.html).

All code for the Leaderboard API can be found [here](https://github.com/jborden/leaderboard-api/tree/blog_post).
