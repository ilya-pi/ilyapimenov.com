---
layout: post
draft: true
title: HyperLogLog with Cascalog
comments: true
categories:
- blog
---

We'll look shortly in how you would utilize awesomeness of both Cascalog and HyperLogLog in order to execute Hadoop M/R tasks with amounts of data too big to have them in their original form.

---

# Intro

<dl>
  <dt><strong>HyperLogLog</strong></dt>
  <dd>Estimator alllowing you to count amount of distinct values of a certain object in environment.</dd>
  <dt><strong>Cascalog</strong></dt>
  <dd>Framework taking the hassle away from you when developing anything on not offer</dd>
</dl>

# Prerequisites

I assume you are already familiar with [Clojure](http://clojure.org/), [Cascalog](https://github.com/nathanmarz/cascalog) framework and [HyperLogLog](http://blog.aggregateknowledge.com/2012/10/25/sketch-of-the-day-hyperloglog-cornerstone-of-a-big-data-infrastructure/) algorithm. 

# Counting Passengers

## Task as stated

We, [Screen6](http://screen6.io/) are located in Amsterdam, and here, in Netherlands, you have this simple and easy system of public transport accessed by OV-Chipkaart and it works like this: prior to accessing the public transport (whether that be a tram, a bus or a train for that matter), you check-in by scanning your card, and once you got to your destination — you check-out.

It's very handy and simple, I use all the time, whenver I'm neither biking, nor walking. 

Now imagine you have all the populous cities and people going threw them on a daily/weekly/monthly/yearly basis, how would you monitor that traffic?

Say, you want to see distribution of passengers in a certain city district (identified by zipcode) over the time; and then you'd love to see that same distribution over a month for two district? three districts? Two cities? 

You cannot just store the list of unique passengers in all the districts per each smallest time-unit — it will be simply way too big to process and access for further analytics; and you cannot just store count of unique passengers, as that information is useless once you come to merging traffic in the different districts and cities (certainly those sets intersect heavily — it is very common to live in Den Haag and work either in Amsterdam, Amsterdam Sloterdijk or Schiphol). 

That is where _"estimators"_ approach comes in handy. It doesn't provide you with exact number, but rathers says what is the cardinality of a certain set with a desired probability, yet being comparative;y dense to the initial set of original items and allowing to merge different register sets; a certain number of algorithms has popped out lately one of them being **HyperLogLog**, and that is exactly the one we'll take our Cascalog-ride with.

## Formatting and sample dataset

So, we assuming that we'll have OV-chipkaartjes access-logs; whenver you've swiped it — we have  a record of `city`, `district`, `tag` and `timestamp`:


| field       | type              | example     |
|:------------|:------------------|:------------|
| `city`      | Varchar           | Amsterdam   |
| `district`  | Zipcode           | 1015        |
| `tag`       | UUID              | 96e4bfec-8cf5-4af1-9469-b7f0dc36dc29 |
| `timestamp` | Ms. from Epoch    | 1363026503  |


To obtain a sample file, we'll throw some clojure code into the REPL:

{% highlight clojure %}
(def cities ["Amsterdam" "Groningen" "Haarlem" "Den Haag" "Utrecht" "Delft" "Edam" ])

(def tags (take 10000 (repeatedly #(str (java.util.UUID/randomUUID)))))

(defn line []
	(let [ city (nth cities (rand-int (.size cities)))
			district (+ (+ 10 (rand-int 89)) (* (+ 10 (rand-int 89)) 100))
			tag (nth tags (rand-int 10000))
			timestamp (+ 1351680873 (rand-int 31536001))]
		(str city " " district " " tag " " timestamp "\n")
	))

(use 'clojure.java.io)
(with-open [wrtr (writer "./ov-chipkaart-accesslogs.txt")]
	(dotimes [n 5000000] (.write wrtr (line))))
{% endhighlight %}

## Defining incoming datasets

Normally you would define one `source` function per dataset type as a simple cascalog query, that does reading, mapping, type checks and conversions. In out example it would be rather simple, we'll simply read the file line-by-line from Hadoop File System:

{% highlight clojure %}

(def ov-fields ["?city" "?district" "?uuid" "?timestamp"])

(defn ov-source "" [directory]
  (let [source (get-tap directory)]
    (c/<- ov-fields
          (source :>> ov-fields))))
{% endhighlight %}

Now we have an `ov-source` cascalog query, which we can utilize as an incoming data stream; without caring to much if the data is correct or not

> **HINT** add filters to the ov-souce query, in order to drop undesired corrupted data.

## Which one?

From all those lib

https://github.com/addthis/stream-lib

## Creating offers

Depending on the case, you might even want to construct an _offer_ string for you hyperloglog value. Yet, keep in mind — it is way better and much more efficient to keep the offers and merge those into the existing hyperloglog value, rahter then merging multiple hyperloglog values.

Lets drop a little code sketch real quick to compare them (though it won't be entirely fair, can you tell why?): 

{% highlight clojure %}
;;
;; single hyperloglog value, multiple inserts
;;
(time 
	(dotimes [n 1000000] 
		(.offer h (str "check" n))))

;;
;; and this is like if we are merging hyperloglogs all the time; 
;; note that we have to create hyperloglog value every time
;;
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

Most of the time, you would have a `source` cascalog query to provide normalized incoming input, and our case wont be any **different**.

## Reducing 

Now, once we have read files, reducing in Cascalog is rather easy and straighforward, yet we will throw in some code in order to deal with hyperloglog values in an easy way:

{% highlight clojure %}
(defmulti merge
          "Merge two HyperLogLog objects.

           If one of values is an offer, then it will be offered to the HyperLogLog
           value.  If both values are offers, new HyperLogLog will be created and both
           values will be offered to this newly created value."
          (fn [left right] [(class left) (class right)]))

(defmethod merge [HyperLogLog HyperLogLog] [left right]
  (.addAll left right)
  left)

(defmethod merge [HyperLogLog String] [left right]
  (.offer left right)
  left)

(defmethod merge [String HyperLogLog] [left right]
  (.offer right left)
  right)

(defmethod merge [nil HyperLogLog] [left right]
  right)

(defmethod merge [HyperLogLog nil] [left right]
  left)

(defmethod merge [String String] [left right]
  (let [acc (create)]
    (.offer acc left)
    (.offer acc right)
    acc))
{% endhighlight %}

As you can see, this will merge whatever you feed it, and provide you with the hyperloglog value.

Now we throw some cascalog glue in:

{% highlight clojure %}
(defn merge-n
  ([h1] h1)
  ([h1 h2] (merge h1 h2))
  ([h1 h2 & more]
   (reduce merge (merge h1 h2) more)))

(defn identity
  {:static true}
  [value]
  (let [res (create)]
    (.offer res value)
    res))

(c/defparallelagg parallel-sum
                  :init-var #'identity
                  :combine-var #'merge-n)

(def sum (ops/each parallel-sum))
{% endhighlight %}

This operations will _"summ up"_ all the offers, the same way you would've use `cascalog.ops/sum` operation on numerical values. In a cascalog query it will end simply up as:

{% highlight clojure %}
    (c/<- [?city ?district ?hll]
      (ov-source :>> ov-fields)
      (hll/sum ?uuid :> ?hll))
{% endhighlight %}          

> todo: ilya — stress out that we are not mergin HLL's among each other

## Persisting

HyperLogLog is an object, basically a set or register set's under the hood, and might you want to persist it — you will require to serialize it somehow.

Now in order to use some sort of JDBC Tap, you'll need to store your bytes sequence, which Cascalog is in a form of string of some kind; without diving too deep, we simply encode it with `base64` and throw it is as an UTF-8 string; so the whole cascalog query will look like this:

{% highlight clojure %}
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

Note that we through in day and year parsing. Now we have nice rows district-day with the amount of passengers see on that day, which can be cheaply merged in the runtime (whenever you request statistics through any online tooling). Sweeet.

### How much would it take?

> todo: ilya — cover up how much would memmory and why would it take to persist current hyperloglog fields in postgress with base64 utf8 encoding

# Gist

> todo: ilya — provide nicelt looking gist for this code

# Links

1. [Clojure: Dynamic programming language that target Java Virtual Machine](http://clojure.org/)
1. [HyperLogLog — Cornerstone of a Big Data Infrastructure](http://blog.aggregateknowledge.com/2012/10/25/sketch-of-the-day-hyperloglog-cornerstone-of-a-big-data-infrastructure/)
1. [Cascalog: Data processing on Hadoop without the hassle](https://github.com/nathanmarz/cascalog)

