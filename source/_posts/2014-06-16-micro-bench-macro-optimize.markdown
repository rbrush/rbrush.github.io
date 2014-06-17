---
layout: post
title: "Micro Benchmarks versus Macro Optimizations in Clojure"
date: 2014-06-16 07:00:00 -0500
comments: true
categories: 
---

Here's a simple question Clojure users hear often:

> What is the overhead of Clojure's persistent data structures?

I ran into this question headlong when profiling and tuning [Clara](https://github.com/rbrush/clara-rules). Clara aims to draw ideas from expert systems into the Clojure ecosystem, making them available as first-class Clojure idioms. Therefore Clara's working memory works like other Clojure data structures: it is immutable and "updates" create a new, immutable working memory.

Anyone skeptical of the benefits of immutability should go watch Rich Hickey's talks like [Simple Made Easy](http://www.infoq.com/presentations/Simple-Made-Easy). Yet these advantages are irrelevant if they don't perform well enough. So we have a challenge: we know that persistent Clojure structures will lose a micro benchmark comparison to mutable counterparts, but can we balance that with macro optimizations made possible with immutability? The answer is _yes_, with the techniques below working for Clara and probably for many other projects.

## Optimizing Clojure Code
Optimizations should have an objective. My objective with Clara was to make performance at least competitive with latest version of Drools, which may be used to solve similar problems. Clara's basis in Clojure offers a number of advantages, but we need to make sure performance isn't a barrier. So I created the [clara-benchmark](https://github.com/rbrush/clara-benchmark) project, using [Criterium](https://github.com/hugoduncan/criterium) to benchmark a number of flows in both Clara and Drools. Some findings:

### It's All About the Algorithms
The first run of profiling didn't look good. Clara was almost ten times slower than Drools for some cases. But it turns out the bulk of this cost had nothing to do with Clojure -- my variation of the Rete algorithm was inefficient, indexing facts for join operations that could never occur due to the nature of the rules. In short, my algorithm sucked.

The good news this was easily exposed with a profiling tool and fixed with little effort. I find algorithms in a language like Clojure to be easier to understand and tune because they are expressed as simple transformations of data. We know the structures we receive and return, and simply need to identify the most efficient way to express the transformation. This is a major contrast to systems that force us keep track of a bunch of additional state when working through the system. 

A better algorithm was the biggest single improvement, bringing Clara within twice Drools performance or better for the use cases tested. But we're not done yet.

### Strongly Isolated Mutability
A better algorithm got us close, but further profiling revealed a bottleneck for some use cases. Rete engines often perform joins over common facts, like this simple example:

```clj
(defquery orders-by-customer-id
  "Returns the orders for the given customer."
  [:?id]
  [Order (= ?id customerId) (= ?total total)]
  [Customer (= ?id id)])
```

This queries the working memory to simply find customer and orders with the same id. (See the [Clara documentation](https://github.com/rbrush/clara-rules/wiki/Guide) for details on use.)

Clara makes extensive use of Clojure's ```group-by``` function to group collections of facts by matching keys. After tuning my algorithm, I discovered that some benchmarks were spending the bulk of their time in ```group-by```. The ```group-by``` implementation can be found in [the Clojure source](https://github.com/clojure/clojure/blob/master/src/clj/clojure/core.clj), but here it is for convenience:

```clj
(defn group-by 
  "Returns a map of the elements of coll keyed by the result of
  f on each element. The value at each key will be a vector of the
  corresponding elements, in the order they appeared in coll."
  {:added "1.2"
   :static true}
  [f coll]  
  (persistent!
   (reduce
    (fn [ret x]
      (let [k (f x)]
        (assoc! ret k (conj (get ret k []) x))))
    (transient {}) coll)))
```

Notice the use of Clojure transients, which are mutable structures designed to be used locally for efficiency. Clojure takes a pragmatic step here. The goal is to keep our systems easy to reason about, and we can achieve that if no external observer can detect mutability. ```group-by``` is a pure function working with immutable structures for all observers, but gains performance by using transients internally.

The trouble I ran into with my benchmarks is that I had many items mapping to the same key. Notice that Clojure's group-by uses a transient map, but that map contains non-transient vectors. So the performance bottleneck arose because this group-by function wasn't "transient enough" for my particular data.

I worked around this by writing an alternate group-by that better fit my needs. Its internals are hideous but are the result of profiling a couple implementations:

```clj
(defn tuned-group-by
  "Equivalent of the built-in group-by, but tuned for when 
   there are many values per key."
  [f coll]
  (->> coll
       ;; Create a mutable map of transient vectors.
       (reduce (fn [map value]
                 (let [k (f value)
                       items (or (.get ^java.util.HashMap map k)
                                 (transient []))]
                   (.put ^java.util.HashMap map k (conj! items value)))
                 map)
               (java.util.HashMap.))
      ;; Make the vectors immutable into a transient map.
      (reduce (fn [map [key value]]
                  (assoc! map key (persistent! value)))
                (transient {}))
      ;; Make the map itself immutable.
      (persistent!)))
```

This is more efficient when there are many items that map to the same key in the returned map, since it uses transient values. (The Java HashMap turned out to be the fastest option to build the result here, but it never escapes this function.) This optimization cut some benchmark times in half. Combined with a number of smaller tweaks, this brought most of Clara's use cases inline with Drools performance. For other use cases Clara significantly outperforms Drools, but we'll get to those later. 

My ```tuned-group-by``` function is faster than Clojure's ```group-by``` for some inputs and slower for others. But this misses a bigger advantage: **_Clojure's philosophy of separating functions and data made swapping implementations trivial, allowing users to pick the right ones for their specific needs._** This isn't so easily done if functions are strongly tied to the data they work with, which is an easy pitfall of object-oriented programming.

### Referential Transparency Breeds Serendipity
Writing functional code tends to create pleasant surprises. We come across significant advantages that wouldn't be possible with a different approach. Considering the following Clara example and it's Drools equivalent:

```clj
(ns clara.benchmark.visit-order-same-day
  (:require [clara.rules :refer :all]
            [clj-time.coerce :as coerce])
  (:import [clara.benchmark.beans Order Visit]))

(defquery same-day-visit
   "Queries orders that occurred the same day as a visit."
   []
   [Order (= ?id customerId) (= ?day (coerce/to-local-date time))]
   [Visit (= ?id customerId) (= ?day (coerce/to-local-date time))])
```

Drools equivalent:

```java
package clara.benchmark.drools.visit_order_same_day;

import clara.benchmark.beans.Order;
import clara.benchmark.beans.Visit;

import org.joda.time.DateTime;

query "same_day_visit"

   Order($id : customerId, $day : time.toLocalDate())
   Visit($id == customerId, $day == time.toLocalDate())

end
```

These rules simply identify things happening on the same day. Yet there is a big difference: Drools does an O(n<sup>2</sup>) Cartesian join where Clara does an O(n*log<sub>32</sub>n) indexed join of each item. Therefore Clara becomes dramatically faster than Drools for large inputs in this case. Also notice how Clara cleanly integrates with Clojure's syntax, as opposed to embedding multiple syntaxes into a file, since it treats [rules as a control structure](http://www.toomuchcode.org/blog/2013/09/24/rules-as-a-control-structure/).

This is possible because of Clojure's emphasis on pure, referentially transparent functions. Since we can replace the function call with its result, we can build an index of that result. The outcome is a significantly more efficient system for this class of problem.

Along the same lines, rule engines facilities to reason over sets of facts can be implemented more efficiently under these constraints. Clara's equivalent of Jess and Drools _accumulators_ simply compile into Clojure [reducers](http://clojure.org/reducers), making them more efficient than the alternatives by simply tapping into that feature.

These advantages arise often: we can defer computation to efficient batch operations. We can transparently spread work across threads without dealing with lock-based concurrency. We can memoize functions or build efficient caches based on fast reference comparison. Importantly, when starting a problem it's not always obvious how these advantages will arise, but these techniques provide an opportunity for great optimizations at the macro level. David Nolen's [work on Om](http://swannodette.github.io/2013/12/17/the-future-of-javascript-mvcs/) is a dramatic example of this in action.

## The Trouble with Benchmarks
X is faster than Y makes for a great incendiary headline on Hacker News, but it doesn't really make sense. X may be faster than Y for workload Z with trade offs A, B, and C...but for some reason those headlines don't get as many upvotes.        

Benchmarks are informative and an important tool to improve our system. But they aren't a real measure of a system's quality or potential. A better measure is how easily a system can be understood, adapted, and expanded. If we can _understand_ the nature of a problem, performance or otherwise, we can usually fix it. Clojure simply provides better mechanisms to understand and improve our systems than other languages I've used.

In short, the trouble with benchmarks is they encourage treating symptoms rather the than the explosion of complexity that limits what we can build.

## Clara's Future
All optimizations discussed here are in master and will be released in Clara 0.6.0 this summer. You can see some current comparisons with Drools in the [clara-benchmark project](https://github.com/rbrush/clara-benchmark). There are still opportunities for improvement in Clara, being a relatively new system. Probably the next significant optimization is greater laziness, [which we're tracking here](https://github.com/rbrush/clara-rules/issues/58). 

Updates will be posted here and on [my twitter feed](https://twitter.com/ryanbrush). I'll also be discussing modern approaches to expert systems, including Clara, at two conferences over the next few months: [Midwest.io](http://www.midwest.io) in July and [strangeloop](https://thestrangeloop.com) in September. 
