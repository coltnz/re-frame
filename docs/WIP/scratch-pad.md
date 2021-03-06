
----------------

## Between 1 and 2

There's a queue. 

When you `dispatch` an event, it is put into a FIFO queue to be processed "vey soon". 

It is important to the design of re-frame that event processing is async. 

On the end of the queue, is a `router` which (very soon) will:
 - pick up events one after the other
 - for each, it extracts `kind` of event (first element of the event vector)
 - for each, it looks up the associated event handler and calls it
  
 
## Between 2 and 3

I lied above.

I said the `router` called the event handler associated with an event.  This is a 
useful simplification, but we'll see in future tutorials that there's more going on.

I'll wave my hands about now and give you a sense of the real story. 

Instead of there being a single handler function, there's actually a pipeline of functions which 
we call an interceptor chain. The handler you write is inserted into the middle of this pipeline.

This function pipeline manages three things:
  - it prepares the `coeffect` for the event handler (the set of inputs required by the handler)
  - it calls the event handler (Domino 2)
  - it handles the `effects` produced by the event handler  (Domino 3)
 

The router actually looks up the associated "interceptor chain", which happens to have the handler wrapped on the end. 

And then it processes the interceptor chain. Which is to say it calls a 


There's 
 - calls the handler , looks at their first , looks at their 
first element, and runs the associated 

Between 1 and 2 it is a queue & router, 
between 2 and 3 it is an interceptor pipeline, and along the 3-4-5-6 domino axis there's a reactive signal graph.  The right 
tool for the job in each case, I'd argue.

While interconnections are critical to how **re-frame works**, 
you can happily **use re-frame** for a long time and be mostly ignorant of their details.

Which is a good thing - back we go to happy brains focusing on the **parts**.


-----


XXX

I'll be using [Reagent] at an intermediate level, so you will need to have done some
introductory Reagent tutorials before going on. Try:
  - [The Introductory Tutorial](http://reagent-project.github.io/) or
  - [this one](https://github.com/jonase/reagent-tutorial) or
  - [Building Single Page Apps with Reagent](http://yogthos.net/posts/2014-07-15-Building-Single-Page-Apps-with-Reagent.html).

##  Implements Reactive Data Flows

This document describes how re-frame implements 
the reactive data flows in dominoes 4 and 5 (queries and views). 

It explains 
the low level mechanics of the process which not something you 
need to know initially. So, you can defer reading and understanding 
this until later, if you wish.  But you should at some point circle
back and grok it.  It isn't hard at all. 
 



## Flow



## Reactive Programming 



We'll get to the meat in a second, I promise, but first one final, useful diversion ...

Terminology in the FRP world seems to get people hot under the collar. Those who believe in continuous-time
semantics might object to me describing re-frame as having FRP-nature. They'd claim that it does something
different from pure FRP, which is true.

But, these days, FRP seems to have become a
["big tent"](http://soft.vub.ac.be/Publications/2012/vub-soft-tr-12-13.pdf)
(a broad church?).
Broad enough perhaps that re-frame can be in the far, top, left paddock of the tent, via a series of
qualifications: re-frame has "discrete, dynamic, asynchronous, push FRP-ish-nature" without "glitch free" guarantees.
(Surprisingly, "glitch" has specific meaning in FRP).

**If you are new to FRP, or reactive programming generally**, browse these resources before
going further (certainly read the first two):
- [Creative Explanation](http://paulstovell.com/blog/reactive-programming)
- [Reactive Programming Backgrounder](https://gist.github.com/staltz/868e7e9bc2a7b8c1f754)
- [presentation (video)](http://www.infoq.com/presentations/ClojureScript-Javelin) by Alan Dipert (co-author of Hoplon)
- [serious pants Elm thesis](https://www.seas.harvard.edu/sites/default/files/files/archived/Czaplicki.pdf)








### React etc.

Okay, so we have some unidirectional, dynamic, async, discrete FRP-ish data flow happening here.

Question: To which ocean does this river of data flow?  Answer: The DOM ocean.

The full picture:
```
app-db  -->  components  -->  Hiccup  -->  Reagent  -->  VDOM  -->  React  --> DOM
```

Best to imagine this process as a pipeline of 3 functions.  Each
function takes data from the
previous step, and produces (derived!) data for the next step.  In the next
diagram, the three functions are marked (f1, f2, f3). The unmarked nodes are derived data,
produced by one step, to be input to the following step.  Hiccup,
VDOM and DOM are all various forms of HTML markup (in our world that's data).

```
app-db  -->  components  -->  Hiccup  -->  Reagent  -->  VDOM  -->  React  -->  DOM
               f1                           f2                      f3
```

In abstract ClojureScript syntax terms, you could squint and imagine the process as:

```Clojure
(-> app-db
   components    ;; produces Hiccup
   Reagent       ;; produces VDOM   (virtual DOM that React understands)
   React         ;; produces HTML   (which magically and efficiently appears on the page).
   Browser       ;; produces pixels
   Monitor)      ;; produces photons?
```


Via the interplay between `ratom` and `reaction`, changes to `app-db` stream into the pipeline, where it
undergoes successive transformations, until pixels colour the monitor you to see.

Derived Data, flowing.  Every step is acting like a pure function and turning data into new data.

All well and good, and nice to know, but we don't have to bother ourselves with most of the pipeline.
We just write the `components`
part and Reagent/React will look after the rest.  So back we go to that part of the picture ...


## Subscribe



`components` (view layer) need to query aspects of `app-db` (data layer).

But how?

Let's pause to consider **our dream solution** for this part of the flow. `components` would:
  * obtain data from `app-db`  (their job is to turn this data into hiccup).
  * obtain this data via a (possibly parameterised) query over `app-db`. Think database kind of  query.
  * automatically recompute their hiccup output, as the data returned by the query changes, over time
  * use declarative queries. Components should know as little as possible about the structure of `app-db`. SQL?  Datalog?

re-frame's `subscriptions` are an attempt to live this dream. As you'll see, they fall short on the declarative
query part, but they comfortably meet the other requirements.

As a re-frame app developer, your job will be to write and register one or more
"subscription handlers" - functions that do a named query.

Your subscription functions must return a value that changes over time (a Signal). I.e. they'll
be returning a reaction or, at least, the `ratom` produced by a `reaction`.

Rules:
 - `components` never source data directly from `app-db`, and instead, they use a subscription.
 - subscriptions are only ever used by components  (they are never used in, say, event handlers).

Here's a component using a subscription:

```Clojure
(defn greet         ;; outer, setup function, called once
  []
  (let [name-ratom  (subscribe [:name-query])]    ;; <---- subscribing happens here
     (fn []        ;; the inner, render function, potentially called many times.
         [:div "Hello" @name-ratom])))
```

First, note this is a [Form-2](https://github.com/Day8/re-frame/wiki/Creating-Reagent-Components#form-2--a-function-returning-a-function)
`component` ([there are 3 forms](https://github.com/Day8/re-frame/wiki/Creating-Reagent-Components)).

Previously in this document, we've used the simplest, `Form-1` components (no setup was required, just render).
With `Form-2` components, there's a function returning a function:
- the returned function is the render function. Behind the scenes, Reagent will wrap this render function
 in a `reaction` to make it produce new Hiccup when its input Signals change.  In our example above, that
 means it will rerun every time `name-ratom` changes.
- the outer function is a setup function, called once for each instance of the component. Notice the use of
 'subscribe' with the parameter `:name-query`. That creates a Signal through which new values are supplied
 over time; each new value causing the returned function (the actual renderer) to be run.

>It is important to distinguish between a new instance of the component versus the same instance of a component reacting to a new value. Simplistically, a new component is returned for every unique value the setup function (i.e. the outer function) is called with. This allows subscriptions based on initialisation values to be created, for example:
``` Clojure
  (defn my-cmp [row-id]
    (let [row-state (subscribe [row-id])]
      (fn [row-id]
        [:div (str "Row: " row-id " is " @row-state)])))
```
In this example, `[my-cmp 1][my-cmp 2]` will create two instances of `my-cmp`. Each instance will re-render when its internal `row-state` signal changes.

`subscribe` is always called like this:

```Clojure
   (subscribe  [query-id some optional query parameters])
```

There is only one (global) `subscribe` function and it takes one parameter, assumed to be a vector.

The first element in the vector (shown as `query-id` above) identifies/names the query and the other elements are optional
query parameters. With a traditional database a query might be:

```
select * from customers where name="blah"
```

In re-frame, that would be done as follows:
   `(subscribe  [:customer-query "blah"])`
which would return a `ratom` holding the customer state (a value which might change over time!).

So let's now look at how to write and register the subscription handler for `:customer-query`

```Clojure
(defn customer-query     ;; a query over 'app-db' which returns a customer
   [db, [sid cid]]      ;; query fns are given 'app-db', plus vector given to subscribe
   (assert (= sid :customer-query))   ;; subscription id was the first element in the vector
   (reaction (get-in @db [:path :to :a :map cid])))    ;; re-runs each time db changes

;; register our query handler
(register-sub
   :customer-query       ;; the id (the name of the query)
   customer-query)       ;; the function which will perform the query
```

Notice how the handler is registered to handle `:customer-query` subscriptions.

**Rules and Notes**:
 - you'll be writing one or more handlers, and you will need to register each one.
 - handlers are functions which take two parameters:  the db atom, and the vector given to subscribe.
 - `components` tend to be organised into a hierarchy, often with data flowing from parent to child via
parameters. So not every component needs a subscription. Very often the values passed in from a parent component
are sufficient.
 - subscriptions can only be used in `Form-2` components and the subscription must be in the outer setup
function and not in the inner render function.  So the following is **wrong** (compare to the correct version above)

```Clojure
(defn greet         ;; a Form-1 component - no inner render function
  []
  (let [name-ratom  (subscribe [:name-query])]    ;; Eek! subscription in renderer
    [:div "Hello" @name-ratom]))
```

Why is this wrong?  Well, this component would be re-rendered every time `app-db` changed, even if the value
in `name-ratom` (the result of the query) stayed the same. If you were to use a `Form-2` component instead, and put the
subscription in the outer functions, then there'll be no re-render unless the value queried (i.e. `name-ratom`) changed.


## The Signal Graph

Let's sketch out the situation described above ...

`app-db` would be a bit like this (`items` is a vector of maps):
```Clojure
(def L  [{:name "a" :val 23 :flag "y"}
        {:name "b" :val 81 :flag "n"}
        {:name "c" :val 23 :flag "y"}])

(def  app-db (reagent/atom  {:items L
                            :sort-by :name}))     ;; sorted by the :name attribute
```

The subscription-handler might be written:

```Clojure
(register-sub
 :sorted-items      ;; the query id  (the name of the query)
 (fn [db [_]]       ;; the handler for the subscription
   (reaction
      (let [items      (get-in @db [:items])     ;; extract items from db
            sort-attr  (get-in @db [:sort-by])]  ;; extract sort key from db
          (sort-by sort-attr items)))))          ;; return them sorted
```


Subscription handlers are given two parameters:

  1. `app-db` - that's a reagent/atom which holds ALL the app's state. This is the "database"
     on which we perform the "query".
  2. the vector originally supplied to `subscribe`.  In our case, we ignore it.

In the example above, notice that the `reaction` depends on the input Signal:  `db`.
If `db` changes, the query is re-run.

In a component, we could use this query via `subscribe`:

```Clojure
(defn items-list         ;; Form-2 component - outer, setup function, called once
  []
  (let [items   (subscribe [:sorted-items])   ;; <--   subscribe called with name
        num     (reaction (count @items))     ;; Woh! a reaction based on the subscription
        top-20  (reaction (take 20 @items))]  ;; Another dependent reaction
     (fn []
       [:div
           (str "there's " @num " of these suckers. Here's top 20")     ;; rookie mistake to leave off the @
           (into [:div ] (map item-render @top-20))])))   ;; item-render is another component, not shown
```

There's a bit going on in that `let`, most of it tortuously contrived, just so I can show off chained
reactions. Okay, okay, all I wanted really was an excuse to use the phrase "chained reactions".

The calculation of `num` is done by a `reaction` which has `items` as an input Signal. And,
as we saw, `items` is itself a reaction over two other signals (one of them the `app-db`).

So this is a Signal Graph. Data is flowing through computation into renderer, which produce Hiccup, etc.

## A More Efficient Signal Graph

But there is a small problem. The approach above might get inefficient, if `:items` gets long.

Every time `app-db` changes, the `:sorted-items` query is
going to be re-run and it's going to re-sort `:items`.  But `:items` might not have changed. Some other
part of `app-db` may have changed.

We don't want to perform this computationally expensive re-sort
each time something unrelated in `app-db` changes.

Luckily, we can easily fix that up by tweaking our subscription function so
that it chains `reactions`:

```Clojure
(register-sub
 :sorted-items             ;; the query id
 (fn [db [_]]
   (let [items      (reaction (get-in @db [:some :path :to :items]))]  ;; reaction #1
         sort-attr  (reaction (get-in @db [:sort-by]))]                ;; reaction #2
       (reaction (sort-by @sort-attr @items)))))                       ;; reaction #3
```

The original version had only one `reaction` which would be re-run completely each time `app-db` changed.
This new version, has chained reactions.
The 1st and 2nd reactions just extract from `db`.  They will run each time `app-db` changes.
But they are cheap. The 3rd one does the expensive
computation using the result from the first two.

That 3rd, expensive reaction will be re-run when either one of its two input Signals change, right?  Not quite.
`reaction` will only re-run the computation when one of the inputs has **changed in value**.

`reaction` compares the old input Signal value with the new Signal value using `identical?`. Because we're
using immutable data structures
(thank you ClojureScript), `reaction` can perform near instant checks for change on even
deeply nested and complex
input Signals. And `reaction` will then stop unneeded propagation of `identical?` values through the
Signal graph.

In the example above, reaction #3 won't re-run until `:items` or `:sort-by` are different
(do not test `identical?`
to their previous value), even though `app-db` itself has changed (presumably somewhere else).

Hideously contrived example, but I hope you get the idea. It is all screamingly efficient.

Summary:
 - you can chain reactions.
 - a reaction will only be re-run when its input Signals test not `identical?` to previous value.
 - As a result, unnecessary Signal propagation is eliminated using highly efficient  checks,
   even for large, deep nested data structures.





Back to the more pragmatic world ...



[SPAs]:http://en.wikipedia.org/wiki/Single-page_application
[SPA]:http://en.wikipedia.org/wiki/Single-page_application
[Reagent]:http://reagent-project.github.io/
[Dan Holmsand]:https://twitter.com/holmsand
[Flux]:http://facebook.github.io/flux/docs/overview.html#content
[Hiccup]:https://github.com/weavejester/hiccup
[FRP]:https://gist.github.com/staltz/868e7e9bc2a7b8c1f754
[Elm]:http://elm-lang.org/
[OM]:https://github.com/swannodette/om
[Prismatic Schema]:https://github.com/Prismatic/schema
[Hoplon]:http://hoplon.io/
[Pedestal App]:https://github.com/pedestal/pedestal-app


<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
## Table Of Contents

<!-- END doctoc generated TOC please keep comment here to allow auto update -->
