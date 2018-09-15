---
layout: post
title: Rules as a Control Structure
comments: false
categories:
---

Rule engines seem to draw love-or-hate reactions. On one hand they offer a simple way to manage lots of arbitrary, complex, frequently-changing business logic. On the other, their simplicity often comes with limitations, and edge cases pop up that can't be elegantly solved in the confines of rules. There are few things more frustrating than a tool meant to help you solve problems actually creates them.

The tragedy is that excellent ideas for modeling logic with rules have been hijacked by a myth: that it's &nbsp;possible to <i>write code</i>&nbsp;-- to unambiguously define logic in a textual form --<i>&nbsp;without actually writing code.</i>&nbsp;We see authoring tools generating rules in limited languages (or XML!), making the case that domain experts can author logic without development expertise. The shortage of good developers makes this tremendously appealing, and this demand has drawn supply.

If you have a limited problem space satisfied by such tools, then great. But problems remain:

* Limited problem spaces often don't stay limited.
* Many problems involving arbitrary domain knowledge are best solved with rules when we can, but require the ability to integrate with a richer programming environment when we must.

So how do we approach this? We need to stop thinking of rule engines as external systems that create artificial barriers between our logic, but as first-class constructs seamlessly integrated in the host language. &nbsp;In other words, rules engines are best viewed as an <i>alternate control structure</i>, suited to the business problem at hand.

Clojure is uniquely positioned to tackle this problem. Macros make sophisticated alternate control structures possible, Clojure's rich data structures make it suitable for solving many classes of problems, and its JVM integration makes it easy to plug into many systems. This is the idea behind <a href="https://github.com/rbrush/clara-rules">Clara</a>, a forward-chaining rules implementation in pure Clojure.

Here's an example from the <a href="https://github.com/rbrush/clara-rules/wiki/Introduction">Clara documentation</a>. &nbsp;In a retail setting with many arbitrary frequently promotions, we might author them like this:

```clj
(defrule free-lunch-with-gizmo
  "Anyone who purchases a gizmo gets a free lunch."
  [Purchase (= item :gizmo)]
  =>
  (insert! (->Promotion :free-lunch-with-gizmo :lunch)))
```

And create a query to retrieve promotions:
```clj
(defquery get-promotions
  "Query to find promotions for the purchase."
  []
  [?promotion <- Promotion])
```

All of this is usable with idiomatic Clojure code:
```clj
(-> (mk-session 'clara.examples.shopping) ; Load the rules.
    (insert (->Customer :vip)
            (->Order 2013 :march 20)
            (->Purchase 20 :gizmo)
            (->Purchase 120 :widget)) ; Insert some facts.
    (fire-rules)
    (query get-promotions))
```

The resulting query returns the matching promotions. More sophisticated examples may join multiple facts and query by parameters; see the <a href="https://github.com/rbrush/clara-rules/wiki/Guide">developer guide</a> or the &nbsp;<a href="https://github.com/rbrush/clara-examples">clara-examples project</a> for more.<br /><br />Each rule constraint and action -- the left-hand and right-hand sides -- are simply Clojure expressions that can contain arbitrary logic. We also benefit from other advantages of Clojure. For instance, Clara's working memory is an immutable, persistent data structure. Some of the advantages of that may come in a later post.<br />

**Rules by domain experts**

So we've broken down some traditional barriers in rule engines, but it seems like this approach comes with a downside: by making rules a control structure in high-level languages, are we excluding non-programmer domain experts from authoring them?<br /><br />We can expand our rule authoring audience in a couple ways:<br /><br />

<div>
<ol>
<li>Encapsulate rules into their own files editable by domain experts, yet compiled into the rest of the system. An audience savvy enough to work with, say, Drools can understand the above examples and many others.</li>
<li>Generate rules from higher-level, domain-specific macros. Business logic could be modeled in a higher-level declarative structure that creates the rules at compile time. Generating rules is actually simpler than most logic generation, since rule ordering and truth maintenance are handled by the engine itself.</li>
<li>Tooling to generate rules directly or indirectly. Like all Lisp code, these rules are also data structures. In fact, they are simpler to work with than an arbitrary s-expression because they offer more structure: a set of facts used by simple constraints resulting in an action, which could also contribute more knowledge to the session's working memory.&nbsp;</li></ol>
</div>
<div>Ultimately, all of this results in Clojure code that plugs directly into a rich ecosystem.&nbsp;</div><div><br /></div><div>These options will be fun to explore, but this isn't my initial target. Let's first create a useful tool for expressing complex logic in Clojure. Hopefully this will become a basis for exploration,&nbsp;borrowing good ideas for expressing business rules and making them available in many environments via the best Lisp available.</div><div><br /></div>

If this seems promising, check out the <a href="https://github.com/rbrush/clara-rules">Clara project on github</a> for more. I'll also post updates on twitter at @ryanbrush.<br /><br />Update: see the <a href="https://groups.google.com/forum/#!topic/clojure/pfeFyZkoRdU">Clojure Google Group</a> for some discussion on this topic.<br /><br />
