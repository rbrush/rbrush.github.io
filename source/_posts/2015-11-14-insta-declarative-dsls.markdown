---
layout: post
title: "Insta-Declarative DSLs!"
date: 2015-11-14 07:47:34 -0600
comments: true
categories:
---

Recently I've focused on [retaking rules for developers](https://www.youtube.com/watch?v=Z6oVuYmRgkk), where rules aim to be a developer-friendly way to untagle complex logic. Yet some problems call for policy changes without the involvement of developers. We need a simple way to write simple rules.

One approach is to offer a minimal Domain-Specific Language focused on the policy our users need to change. In this post we take a simple example, write a DSL, parse it, validate it, and run it against some data. We'll use the excellent [Instaparse](https://github.com/Engelberg/instaparse) library to define the grammar and create the parse tree, and convert that tree into rules executable with [Clara](http://www.clara-rules.org).

First we figure out how we want our DSL to look. To keep things simple, let's imagine a retail setting and let our business user define promotions and discounts based on customer and order information. An example might look like this:

```clj
discount my-discount 15 when customer status is platinum.
discount extra-discount 10 when customer status is gold and total value > 200.
promotion free-widget-month free-widget when customer status is gold and order month is august.
```

Rule engines are a good fit for declarative DSLs because rule engines *themselves* are declarative. We can see the rule-like structure in the above example: apply this policy when that set of conditions is true.

Now we need to write a function that converts our friendly DSL into rules we can run. Fortunately, in Clara [rules are data](/blog/2014/01/19/rules-as-data/), so our function needs to produce a simple data structure rather than generating rules using string manipulation. Using Clojure and [Prismatic Schema](https://github.com/Prismatic/schema) to define the structure, our function looks like this:

```clj
(s/defn load-user-rules :- [clara.rules.schema/Production]
  "Converts a business rule string into Clara productions."
  [business-rules :- s/Str]
  ;; TODO: convert DSL to Clara rule structures.
  )
```

So let's implement it! First we use [Instaparse](https://github.com/Engelberg/instaparse) to define our grammar. We can start with the major productions and break their contents. So the discount production would look like this:

```
DISCOUNT = <'discount'> NAME PERCENT <'when'> CONDITION [<'and'> CONDITION]* <'.'>;
```

And it contains a series of conditions, like this:

```
CONDITION = FACTTYPE FIELD OPERATOR VALUE ;
```

And so on. Here is the complete grammar we will use, which we simply bring into our Clojure session:

```clj
(require '[instaparse.core :as insta])

(def shopping-grammar
  (insta/parser
   "<RULES> = [DISCOUNT | PROMOTION]+
    PROMOTION = <'promotion'> NAME PROMOTIONTYPE <'when'> CONDITION [<'and'> CONDITION]* <'.'>;
    DISCOUNT = <'discount'> NAME PERCENT <'when'> CONDITION [<'and'> CONDITION]* <'.'>;
    <PERCENT> = NUMBER ;
    PROMOTIONTYPE = STRING ;
    <NAME> = STRING ;
    NUMBER = #'[0-9]+' ;
    <STRING> = #'[A-Za-z][A-Za-z0-9_-]+' ;
    CONDITION = FACTTYPE FIELD OPERATOR VALUE ;
    FACTTYPE = 'customer' | 'total' | 'order' ;
    <FIELD> = STRING ;
    OPERATOR = 'is' | '>' | '<' | '=' ;
    <VALUE> = STRING | NUMBER ;
    "
   :auto-whitespace :standard))
```   

The of angle brackets indicate productions to omit from the abstract syntax tree and replace by their children. This isn't strictly necessary, but simplifies things when transform the tree.

The ```insta-parser``` function actually returns a function that converts an input to the syntax tree! So we can just call it with our DSL and pretty-print the results:

```clj
(clojure.pprint/pprint
 (shopping-grammar
  "discount my-discount 15 when customer status is platinum.
   discount extra-discount 10 when customer status is gold and total value > 200.
   promotion free-widget-month free-widget when customer status is gold and order month is august."))
```

Which produces this:

```clj
([:DISCOUNT
  "my-discount"
  [:NUMBER "15"]
  [:CONDITION
   [:FACTTYPE "customer"]
   "status"
   [:OPERATOR "is"]
   "platinum"]]
 [:DISCOUNT
  "extra-discount"
  [:NUMBER "10"]
  [:CONDITION [:FACTTYPE "customer"] "status" [:OPERATOR "is"] "gold"]
  [:CONDITION
   [:FACTTYPE "total"]
   "value"
   [:OPERATOR ">"]
   [:NUMBER "200"]]]
 [:PROMOTION
  "free-widget-month"
  [:PROMOTIONTYPE "free-widget"]
  [:CONDITION [:FACTTYPE "customer"] "status" [:OPERATOR "is"] "gold"]
  [:CONDITION [:FACTTYPE "order"] "month" [:OPERATOR "is"] "august"]])
```

Just for fun, let's see what happens when a user mistypes some input. Let's say "customer" is misspelled when we evaluate the input against our grammar. So running this:

```clj
(println
 (shopping-grammar
  "discount my-discount 15 when customeer status is platinum."))
```

prints out this:

```
Parse error at line 1, column 30:
discount my-discount 15 when customeer status is platinum.
                             ^
Expected one of:
order
total
customer
```

Great! The error is pretty clear and gives the user options how to fix it. Instaparse does a great job at this.

Alright, let's get back on track and imagine our user fixed the error. We now have a nice parse tree...we just need to convert it into rules. One way to do this is write a ```map``` function that goes through each top-level production and returns a Clara rule. This is a fine approach, and may be a better fit depending on the transformation needed. But in this case I'm going to take advantage of another feature of Instaparse: the ability to apply arbitrary transformations to productions in the tree.

The simplest example is we want to replace productions like [:NUMBER "15"] with...the actual number 15. This tends to be useful for things like, you know, math.

So let's run a production through our grammar and use the ```insta/transform``` function to take a map of transformation for productions. We use Clojure's threading macro to make wiring functions together more readable:

```clj
(->> "discount my-discount 15 when customer status is platinum."
     (shopping-grammar)
     (insta/transform {:NUMBER #(Integer/parseInt %)})
     (clojure.pprint/pprint))
```

This transforms our tree into this, where we have a number rather than an AST production:

```clj
([:DISCOUNT
  "my-discount"
  15
  [:CONDITION
   [:FACTTYPE "customer"]
   "status"
   [:OPERATOR "is"]
   "platinum"]])
```   

So we've taken our first step of transforming our tree into an actual, executable rule! Now we need to do some more transformations:

* ```:OPERATOR``` gets transformed to a Clojure comparison function
* ```:FACTTYPE``` gets transformed to a Clojure type. In this case we just use Clojure records.
* ```:PROMOTIONTYPE``` is an enumeration, which we idiomatically transform to a Clojure keyword
* ```:CONDITION``` gets transformed into the left-hand side expression of a rule
* ```:DISCOUNT``` and ```:PRODUCTION``` get transformed into actual Clara rules, built on the transformations above! These match the ```clara.rules.schema/Production``` Prismatic Schema.

I find it's best to build this type of logic from the bottom up in a REPL or a REPL-connected editor. Just start with the simplest transformations, like ```:NUMBER``` and ```:OPERATOR```, make sure they work in the REPL, then work on the transformations that use them. I also found myself tweaking the grammar to omit or hide unnecessary productions. After a few quick iterations I ended up with this:

```clj
(def operators {"is" `=
                ">" `>
                "<" `<
                "=" `=})

(def fact-types
  {"customer" Customer
   "total" Total
   "order" Order})

(def shopping-transforms
  {:NUMBER #(Integer/parseInt %)
   :OPERATOR operators
   :FACTTYPE fact-types
   :CONDITION (fn [fact-type field operator value]
                {:type fact-type
                 :constraints [(list operator (symbol field) value)]})

   ;; Convert promotion strings to keywords.
   :PROMOTIONTYPE keyword

   :DISCOUNT (fn [name percent & conditions]
               {:name name
                :lhs conditions
                :rhs `(insert! (->Discount ~name ~percent))})

   :PROMOTION (fn [name promotion-type & conditions]
                {:name name
                 :lhs conditions
                 :rhs `(insert! (->Promotion ~name ~promotion-type))})})
```

That's it! This works because the transformations build on top of lower-level transformations. For instance, the ```:CONDITION``` transformation is given a fact-type and an operator because those were transformed by the ```:FACTTYPE``` and ```:OPERATOR``` transformations, respectively. Users could choose to leave out lower-level transformations and have ```:CONDITION``` do all of the work, but the above approach shows the power of this Instaparse feature.

Also note our use of the Clojure syntax quote (`) and unquote (~). These are typically used when writing Macros, but they're convenient in this case to build expressions that Clara turns into rules. (After all, Clara is really just a big macro that converts user expressions into a rete network!)

Now let's run this set of transformations against our input data:

```clj
(->> "discount my-discount 15 when customer status is platinum.
      discount extra-discount 10 when customer status is gold and total value > 200.
      promotion free-widget-month free-widget when customer status is gold and order month is august."
     (shopping-grammar)
     (insta/transform shopping-transforms)
     (clojure.pprint/pprint))
```

This produces:

```clj
({:name "my-discount",
  :lhs
  ({:type clara.examples.insta.Customer,
    :constraints [(clojure.core/= status "platinum")]}),
  :rhs
  (clara.rules/insert!
   (clara.examples.insta/->Discount "my-discount" 15))}
 {:name "extra-discount",
  :lhs
  ({:type clara.examples.insta.Customer,
    :constraints [(clojure.core/= status "gold")]}
   {:type clara.examples.insta.Total,
    :constraints [(clojure.core/> value 200)]}),
  :rhs
  (clara.rules/insert!
   (clara.examples.insta/->Discount "extra-discount" 10))}
 {:name "free-widget-month",
  :lhs
  ({:type clara.examples.insta.Customer,
    :constraints [(clojure.core/= status "gold")]}
   {:type clara.examples.insta.Order,
    :constraints [(clojure.core/= month "august")]}),
  :rhs
  (clara.rules/insert!
   (clara.examples.insta/->Promotion
    "free-widget-month"
    :free-widget))})
```         

Now we have a sequence of rules we can run! We can pass this directly into the [mk-session](http://www.clara-rules.org/apidocs/0.9.0/clojure/clara.rules.html#var-mk-session) function and create an actual rule session!

We can also combine these rules with others written by Clara's [defrule](http://www.clara-rules.org/apidocs/0.9.0/clojure/clara.rules.html#var-defrule) or generated from some other source. You can see the full code in the [clara.examples.insta](https://github.com/rbrush/clara-examples/blob/master/src/main/clojure/clara/examples/insta.clj) namespace in the clara-examples project, but here is the pertinent segment for running our rules:

```clj
(let [session (-> (mk-session 'clara.examples.insta (load-user-rules example-rules))
                  (insert (->Customer "gold")
                          (->Order 2013 "august" 20)
                          (->Purchase 20 :gizmo)
                          (->Purchase 120 :widget)
                          (->Purchase 90 :widget))
                  (fire-rules))]

  (clojure.pprint/pprint (query session get-discounts))
  (clojure.pprint/pprint (query session get-promotions)))
```

Running this produces the following output:

```clj
({:?discount {:name "extra-discount", :percent 10}})
({:?discount {:reason "free-widget-month", :type :free-widget}})
```

And that's it! The complete code for this is in [clara-examples](https://github.com/rbrush/clara-examples/blob/master/src/main/clojure/clara/examples/insta.clj). Details on Instaparse can be found on the [Instaparse github page](https://github.com/Engelberg/instaparse) and Clara documentation is at [clara-rules.org](http://www.clara-rules.org). You can also reach me on twitter [@ryanbrush](https://twitter.com/ryanbrush).

Finally, we once again see how powerful Clojure's composable design is. The Instaparse and Clara libraries were built completely independently, but since both use functional transformations of immutable data structures we were able to combine them to create something useful in a small amount of code. Plus hacking on this stuff is just plain *fun*.

UPDATE: I posted an answer to a follow up question to [this thread in the Clara Google group](https://groups.google.com/forum/#!topic/clara-rules/ME7CKVdBMqg). If there are other topics, please feel free to use that thread or create a new one.
