---
layout: post
author: <a href="http://ilyapimenov.com/">Ilya Pimenov</a>
title: HyperLogLog with Cascalog
comments: true
categories:
- blog
---
 _Updated: 22 nov 2013_

We'll look briefly in how you would utilize awesomeness of both Cascalog and HyperLogLog in order to execute Hadoop M/R tasks with amounts of data too big to have them in their original form.

---

# Intro

<dl>
  <dt><strong>HyperLogLog</strong></dt>
  <dd>Cardinality estimator allowing you to count amount of distinct values.</dd>
  <dt><strong>Cascalog</strong></dt>
  <dd>The main use cases for Cascalog are processing "Big Data" on top of Hadoop or doing analysis on your local computer. Cascalog is a replacement for tool like Pig, Hive, and Cascading which operates at a significantly higher level of abstraction than those tools.</dd>
</dl>

# Prerequisites

We'll assume you are already familiar with the [Clojure](http://clojure.org/), [Cascalog](https://github.com/nathanmarz/cascalog) framework and [HyperLogLog](http://blog.aggregateknowledge.com/2012/10/25/sketch-of-the-day-hyperloglog-cornerstone-of-a-big-data-infrastructure/) algorithm. Otherwise — jump straight to the bottom for some links, there are some nice reads there.

# Counting Passengers

## Task as stated

We'll deal with an example that is similar in its mechanics with a task we face ourselves at [Screen6](http://screen6.io/), yet being from another market — passenger transportation.

We are located in Amsterdam, and here, in Netherlands, you have this simple and easy system of public transport accessed by NFC card and which works as follows: prior to accessing the public transport (whether that be a tram, a bus or a train for that matter), you check-in by scanning your card, and once you get to your destination — you check-out with that same NFC card (which is called "OV-Chipkaart").

It's convenient, I use all the time, whenever I'm neither biking or walking.

Now imagine you have all these populous cities and people commuting between them on a daily/weekly/monthly/yearly basis, how would you monitor that traffic?

Say, you want to see distribution of passengers in a certain city district (identified by zipcode) over time, and on top of that you'd love to see that same distribution over a month for two districts? Or three districts? Or two cities altogether?

You can just store all events without any aggregations and directly do queries on that dataset, but it possesses two issues within itself: perfomance concern, as we are dealing with what supposed to be a very big amount of data, and hence — persistence issues.

You cannot just store the list of unique passengers in all the districts per each smallest time-unit — it will be simply way too big to process and access for further analytics; and you cannot just store count of unique passengers, as that information is useless once you come to merging traffic in the different districts and cities (certainly those sets intersect heavily — it is very common to live in The Hague and work either in Amsterdam, Amsterdam Sloterdijk or Schiphol).

That is where a _caridnality estimator_ comes in handy. It doesn't provide you with an exact number, but rather tells you what the cardinality of a certain set is with a desired error margin, yet being comparatively dense to the initial set of original items and allowing you to merge different sets.

A certain number of algorithms has popped out lately one of them being **HyperLogLog**, and that is the one we are taking with us on our Cascalog-ride.

## Formatting and sample dataset

We are assuming that we have OV-Chipkaart access-logs: each and every swipe (Log file line) gives you a record of `city`, `district`, `tag` and `timestamp`:


| field       | type              | example     |
|:------------|:------------------|:------------|
| `city`      | Varchar           | Amsterdam   |
| `district`  | Zipcode           | 1015        |
| `tag`       | UUID              | 96e4bfec-8cf5-4af1-9469-b7f0dc36dc29 |
| `timestamp` | Ms. from Epoch    | 1363026503  |

<p style="text-align: center;">Table 1. OV-Chipkaart access log record format</p>

Now, we didn't get any logs from OV-Chipkaarts proprietor (though, I would've been delighted to get my hands on them), so we will have to generate some randomized data them ourselves. Here is the clojure code snippet code that will do just that:

{% highlight clojure %}
(def cities ["Amsterdam" "Groningen" "Haarlem"
  "Den-Haag" "Utrecht" "Delft" "Edam" ])

(def tags (repeatedly 10000 #(str (java.util.UUID/randomUUID))))

(defn line []
  (let [city (rand-nth cities)
        district (+ (+ 10 (rand-int 89)) (* (+ 10 (rand-int 89)) 100))
        tag (rand-nth tags)
        timestamp (+ 1351680873 (rand-int 31536001))]
        (str city " " district " " tag " " timestamp "\n")))

(use 'clojure.java.io)
(with-open [wrtr (writer "./ov-chipkaart-accesslogs.txt")]
  (dotimes [_ 5000000] (.write wrtr (line))))
{% endhighlight %}

This would provide us with a file of approximate 157Mb size holding 5.000.000 OV-chipkaart accesslog records; gziped — 78Mb.

## Defining incoming datasets

Normally you would define one `source` function per dataset type as a simple cascalog query, that does reading, mapping, type checks and conversions. Our example won't be any different — we'll read the file line-by-line from Hadoop File System (which will nicely pickup both gzippped and not gzipped files for us):

{% highlight clojure %}
(ns ...
  (:require ...
            [cascalog.api :as c]
            [cascalog.ops :as ops]))

(def ov-fields ["?city" "?district" "?uuid" "?timestamp"])

(defn ov-source "" [directory]
  (let [source (get-tap directory)]
    (c/<- ov-fields
          (source :>> ov-fields))))
{% endhighlight %}

Now we have an `ov-source` cascalog query, which can be utilized as an incoming data stream; without caring too much if the data is correct or not.

<div class="note">Add filters to the ov-source query, in order to drop undesired corrupted data.</div>

## Which one?

Now, you can either go and implement your own implementation of HyperLogLog that will suit your needs, or you can pick one of the following already existing opensourced implementatoins:

| Company  |  Library       | Link              | Language | License |
|:---------|:---------------|:------------------|:---------|:--------|
| Facebook | [JCommon](https://github.com/facebook/jcommon/) | [HyperLogLog.java](https://github.com/facebook/jcommon/blob/master/stats/src/main/java/com/facebook/stats/cardinality/HyperLogLog.java) | Java | Apache License 2.0 |
| Twitter  | [Algebird](https://github.com/twitter/algebird/) | [HyperLogLog.scala](https://github.com/twitter/algebird/blob/develop/algebird-core/src/main/scala/com/twitter/algebird/HyperLogLog.scala) | Scala | Apache License 2.0 |
| AddThis | [Stream-Lib](https://github.com/addthis/stream-lib) | [HyperLogLog.java](https://github.com/addthis/stream-lib/blob/master/src/main/java/com/clearspring/analytics/stream/cardinality/HyperLogLog.java) | Java | Apache License 2.0 |
| indie! | [yukim's gists](https://gist.github.com/yukim) | [HyperLogLog.java](https://gist.github.com/yukim/2597943#file-hyperloglog-java) | Java | Apache License 2.0 |

<p style="text-align: center;">Table 2. Available HyperLogLog implementations</p>

We chose [AddThis](http://www.addthis.com/)'s [Stream-Lib](https://github.com/addthis/stream-lib)'s implementation, as from my subjective point of view it seemed to be most clear, nicely documented and reasonably implemented; besides, they added a bunch of other sweet things for cardinality estimation in that same library, together with the list of papers their implementations were based upon.

## Creating offers

There are two approaches to merging HyperLogLog values:

1. Either you create an object wrapping up each value in a HyperLogLog value and merge those after
1. Or you create HyperLogLog value once for a set of values and then _offer_ them to this particular object

Depending on the case, you might even want to construct an _offer_ string for your HyperLogLog value as composite key of multiple values in the row. But no matter what you do, keep in mind — it is way better and much more efficient to keep the offers and merge those into the existing HyperLogLog value, rather than merging multiple HyperLogLog values.

Lets drop a little code sketch real quick to compare them:

{% highlight clojure %}
(ns ...
  (:import [com.clearspring.analytics.stream.cardinality HyperLogLog HyperLogLog$Builder]))

;; single hyperloglog value, multiple inserts
(time
  (dotimes [n 1000000]
    (.offer h (str "check" n))))

;; and this is like if we are merging hyperloglogs all the time;
;; note that we have to create hyperloglog value every time
(time
  (dotimes [n 1000000]
    (merge h (let [nh (create)]
      (.offer nh (str "check" n))
      nh))))
{% endhighlight %}

Here is a sample time comparison `offer` vs `merge` result:

| operation | execution 1    | execution 2    | execution 3    | execution 4    |
|:---------:|:--------------:|:--------------:|:--------------:|:--------------:|
| `offer`   | 1513.87 msecs  | 1479.377 msecs | 1493.119 msecs | 1485.23 msecs  |
| `merge`   | 7433.302 msecs | 7306.165 msecs | 7369.459 msecs | 7246.366 msecs |

<p style="text-align: center;">Table 3. HyperLogLog offer vs merge execution times</p>

<div class="note">Store list of raw distinct values and offer them to a single HyperLogLog object instead of merging separate HyperLogLog values, when possible.</div>

## Reducing

Now, once we have read files, reducing in Cascalog is rather easy and straightforward, yet we will throw in some code in order to deal with HyperLogLog values in an easy way:

{% highlight clojure %}
(defprotocol IHyperLogLogMerge 
  (hyperloglog-val [this])
  (merge [this other])
  (merge-with-hyperloglog [this other-hll]))
 
(extend-protocol IHyperLogLogMerge
  nil
  (hyperloglog-val [this] nil)
  (merge-with-hyperloglog [this other-hll] other-hll)
  (merge [this other] other)
 
  Object
  (hyperloglog-val [this] (doto (create) (.offer this)))
  (merge-with-hyperloglog [this other-hll] (.offer other-hll this) other-hll)
  (merge [this other]
    (merge (hyperloglog-val other) this))
 
  HyperLogLog
  (hyperloglog-val [this] this)
  (merge-with-hyperloglog [this other-hll] (.addAll this other-hll) this)
  (merge [this other]
    (merge-with-hyperloglog other this)))
{% endhighlight %}

As you can see, this will merge whatever you feed it, and provide you with the HyperLogLog value. Though, not used in this particular example, will be used extensively in reallife scenario, when aggregating already obtainded aggregated results (that will hold not offers, but HyperLogLog values). 

And then finally, let's throw in some glue to aggregate HyperLogLog values in the Cascalog queries:

{% highlight clojure linenos %}
(c/defaggregateop sum*
  ([] (create))
  ([state val]
    (.offer state val)
    state)
  ([state] [state]))
  
(def sum
  (ops/each sum*))
{% endhighlight %}

This operations will _"sum up"_ all the offers, the same way you would've use `cascalog.ops/sum` operation on numerical values, you can use this operation on HyperLogLog values. In a cascalog query it will end up simply as:

{% highlight clojure %}
(c/<- [?city ?district ?hll]
  (ov-source :>> ov-fields)
  (hll/sum ?uuid :> ?hll))
{% endhighlight %}

## Persistent

HyperLogLog is an object, basically a set or register set's under the hood, and might you want to persist it — you will require to serialize it somehow.

Now in order to use some sort of Tap, you'll need to store your bytes sequence, which in case of Cascalog is in a form of string of some kind; since at the moment we use both JDBC Taps and plain CSV files, without diving too deep, we simply encode it with `base64` and throw it is as an UTF-8 string; so the whole cascalog query will look like this:

{% highlight clojure linenos %}
(c/defmapop stringify [hll-object]
  [(Base64/encodeBase64String (.getBytes hll-object))])

(defn get-day-n-year [epoch-time]
  (let [ epoch-time-long (Long/parseLong epoch-time)
         in-millis (* epoch-time-long 1000)
         date (time2/from-long in-millis)]
    [(.getDayOfYear date) (.getYear date)]))

(defn count-gvb-passengers [path]
  (let [ov-source (ov-source path)]
    (c/<- [?city ?district ?day ?year ?cardinality ?base64-hll]

      (:trap (c/hfs-textline "/tmp/hll-demo-errors" :sinkmode :replace ))

      (ov-source :>> ov-fields)

      (hll/sum ?uuid :> ?hll)
      (cardinality ?hll :> ?cardinality)
      (get-day-n-year ?timestamp :> ?day ?year)
      (hll/stringify ?hll :> ?base64-hll))))
{% endhighlight %}

Note that we throw in day and year parsing.

Now we have nice rows grouped by `(district, day)` key with the amount of passengers seen that day, which can be cheaply merged in the runtime (whenever you request statistics through any online tooling).

Sweeet.

# Gist

And, of course, here is the full gist, of the code I used to demo HyperLogLog with Cascalog in this article; in order to run:

{% highlight bash %}
  $ git clone https://gist.github.com/7319327.git
  $ cd cd ./7319327/
  $ lein demo ./ov-chipkaart-accesslogs.txt
{% endhighlight %}

It will produce output like:

| ?city | ?district | ?day | ?year | ?cardinality |
|:------|:---------:|:----:|:-----:|:------------:|
| Amsterdam | 6427 | 327 | 2012 | 1 |
| Amsterdam | 6427 | 328 | 2012 | 3 |
| Delft | 2290 | 53 | 2013 | 1 |
| Delft | 3855 | 109 | 2013 | 1 |
| Delft | 4091 | 74 | 2013 | 1 |
| Delft | 5589 | 200 | 2013 | 1 |
| Den-Haag | 3797 | 300 | 2013 | 1 |
| Edam | 3886 | 47 | 2013 | 1 |
| Edam | 5594 | 272 | 2013 | 1 |
| Haarlem | 1134 | 108 | 2013 | 1 |
| Haarlem | 4584 | 220 | 2013 | 1 |

<p style="text-align: center;">Table 4. Demo output</p>

{% gist 7319327 %}

# Links

1. [Clojure: Dynamic programming language that target Java Virtual Machine](http://clojure.org/)
1. [HyperLogLog — Cornerstone of a Big Data Infrastructure](http://blog.aggregateknowledge.com/2012/10/25/sketch-of-the-day-hyperloglog-cornerstone-of-a-big-data-infrastructure/)
1. [Cascalog: Data processing on Hadoop without the hassle](https://github.com/nathanmarz/cascalog)
