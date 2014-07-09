---
layout: post
title:  Building a RESTful Clojure App on Heroku
date:   2014-07-08
published: true
---

*This is based on a talk I gave at the [July SF Clojure Meetup](http://www.meetup.com/The-Bay-Area-Clojure-User-Group/events/148824482/).*

I've not read much about building real Clojure applications. Not because I don't want to, but because there's not much written. You can find plenty of examples of how to do particular things, but nothing like case studies that present how someone put together a whole program. This pattern holds in the literature of most programming languages, but I notice it more in Clojure because there is a lot less written about how to engineer software in functional languages. Although many of the same design patterns popular in imperative and object-oriented languages carry over in theory, the implementation differs significantly in some cases as to be unrecognizable.

In that vein, I wanted to share how my first full-fledged Clojure app came together. I've previously written Clojure in the context of Storm and Pallet, so I was always filling in details to do what I wanted within someone else's architecture. Now that I was going to build an entire application from scratch, I needed a better understanding of all aspects of code and project structure in Clojure.

The application in question is an ad conversion tracker for my employer, [AdStage](http://www.adstage.io/). The purpose of the conversion tracker is to offer an HTTP interface that accepts tracking event information from JavaScript code running on our customer's sites. When a user completes some action and converts (like making a purchase or signing up for a newsletter), if they previously clicked on an ad with AdStage conversion tracking on it we want to count that conversion and attribute it to the ad. This application is responsible both for collecting the tracking info and processing it to return conversion counts.

The basic workflow of the application is that it accepts tracking info in an HTTP request (which we refer to as a hit), quickly puts the hit on a queue, and then asynchronously processes these hits to see if they match any of our customer's conversion goals. If they do, we count the hit as a conversion and attribute the conversion to the ad or ads that led to it.

In our case we decided to build an application we would deploy on Heroku, and I chose Clojure mostly because I like it. We would integrate hit collection, hit processing, and conversion reporting into a single app for now, but also keep in mind that in the future we might need to split parts of it out to get better performance or meet new business needs. We chose RabbitMQ and Cassandra as the underlying services to support queueing and persistent data storage since RabbitMQ was easy to setup and gave us decent headroom and Cassandra is already heavily used elsewhere in our products.

Since the app needs to respond to HTTP requests, both for collecting hits and interacting with our other services, it naturally made sense to start with [Ring](https://github.com/ring-clojure/ring) and [Compojure](https://github.com/weavejester/compojure). Heroku made this choice even easier by providing a handy [getting started guide and template](https://devcenter.heroku.com/articles/getting-started-with-clojure) that make use of these libraries. This presented the first opportunity for making architecture choices: how would routes translate into computations?

The decision was to follow a pattern of building namespaces that represent the resources each route addresses. So for collecting hits on the `/collect` route we have the `adstage.conversion-tracker.resource.collect` namespace that contains functions describing actions the collection resource should perform. It looks a little something like this:

```clojure
(ns adstage.conversion-tracker.resource.collect
  (:require [clojure.data.json :as json]
            [clojurewerkz.cassaforte.uuids :as uuids]
            [clj-time.core :as t]
            [clj-time.coerce :as ct]
            [adstage.conversion-tracker.queue.new-hit :as new-hit]
            [adstage.conversion-tracker.queue.to-s3 :as to-s3]))

(defn create! [request]
  (let [params (:params request)
        headers (:headers request)
        user-agent (get headers "user-agent")
        referer (get headers "referer")
        user-ip (:remote-addr request)
        hit-id (str (uuids/random))
        ts (or (try (ct/from-long (Long. (:ts params)))
                    (catch Exception e))
               (t/now))
        hit-data (assoc params :hit-id hit-id
                               :user-agent user-agent
                               :referer referer
                               :ts ts
                               :user-ip user-ip)]
    (new-hit/send! hit-data)
    (to-s3/send! hit-data)
    {:status 200}))
```

And gets called from a Compojure route like this:

```clojure
  (GET "/collect" [:as request]
       (as-collect/create! request))
  (POST "/collect" [:as request]
        (as-collect/create! request))
```

Those calls in `create!` to `new-hit/send!` and `to-s3/send!` are what puts the hit on the queue. The queue is isolated from the rest of the application in a namespace and each queue provides a common interface that doesn't know anything about its underlying implementation. This is to facilitate isolation during tests, allow the underlying queueing service to be changed without changing business logic (we will eventually need to move to Kafka to handle greater loads), and make it relatively easy to later break the queues out into a separate service if need be.

How isolated? Just take a look at the code:

```clojure
(ns adstage.conversion-tracker.queue.new-hit
  (:require [adstage.util :refer :all]
            [adstage.conversion-tracker.queue.core :as queue]))

(def queue-name "hit.new")
(def queue-config [:durable true
                   :auto-delete false
                   :exclusive false])
(def queue-exchange "")
(defenv hit-qos 1)

(defn send! [hit-data]
  (queue/send-message! queue-name queue-exchange hit-data))

(defn process! [processing-fn error-fn]
  ; this function should block the thread consuming
  ; messages off the queue forever
  (queue/process-messages! queue-name hit-qos processing-fn error-fn))

(defn get-next! []
  (queue/get-next-message! queue-name))

(defn metadata []
  (queue/get-metadata queue-name))
```

Essentially all functionality is implemented by `queue.core` with `queue.new-hit` providing an interface to it. As far as the rest of the application knows, `new-hit` is just a queue that it can send things to and get things from. This is the kind of separation and isolation you may be used to seeing achieved with objects, but here we do it with namespaces instead.

And this is not even that great an implementation! It can be further cleaned up to remove boilerplate code, since every queue implements this same functionality with different configuration. The conversion to functional style is not complete, because here I'm treating namespaces too much like classes. That's exactly the kind of trouble I ran into, though, building our conversion tracker: I had lots of experience building things with objects but not a lot using modern functional languages, and few good examples of what to do instead.

Of course there is more to the application that I don't have time to go into now, but to at least leave you with something of the big picture, here's an architectural diagram I drew up after building the conversion tracker:

![Conversion Tracker Architectural Diagram](/assets/conversion-tracker-architectural-diagram.png)

As you can see, we've only skimmed the surface, and we haven't even looked at the tests. In the future I'll try to write more about these topics both as I improve the conversion tracker and write additional Clojure applications.

And speaking of writing additional Clojure applications, Clojure is gaining traction at AdStage. In addition to the conversion tracker, we're now rewriting our ad metrics service in Clojure, continuing to use Pallet for server automation, and have future plans to rewrite additional services in Clojure to replace the current Ruby implementations. Sound like fun? [We're hiring!](http://www.adstage.io/jobs)
