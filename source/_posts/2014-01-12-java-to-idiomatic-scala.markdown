---
layout: post
title: "Java to idiomatic scala"
date: 2014-01-12 12:55
comments: true
categories:
published: false
---

Here's the scenario. I've got a simple caching interface and a couple of concrete implementations. One is a local cache using an LRU map and the other is a distributed cache backed by Redis.

``` scala
trait Cache {

  def get[A <: Serializable](key: String): Option[A]

  def set(key: String, value: Serializable)
}
```

Redis may be fast, but it's not as fast as a local cache. After all, data still has to be deserialized and sent across a network from a Redis server. But local memory is limited and not available to other machines in a cluster.

The best of both worlds can be had by putting the two caches together in a fall through scenario. Basically, prefer the local cache and fall back on Redis. The composite pattern should do nicely.

``` scala
class CompositeCache(caches: List[Cache] = List()) extends Cache {

  def add(cache: Cache) = new CompositeCache(caches :+ cache)

  def get[A <: Serializable](key: String): Option[A] = {

    case class LazyGet(cache: Cache) {lazy val value = cache.get[A](key)}

    val (miss, hit) = caches map LazyGet span (_.value.isEmpty)

    hit.headOption flatMap {
      c =>
        miss foreach (_.cache.set(key, c.value.get))
        c.value
    }
  }

  def set(key: String, value: Serializable) {
    caches foreach (_.set(key, value))
  }
}
```

The `span` operator is basically the same as doing `dropWhile` and `takeWhile` at the same time and putting the result in a tuple. The important characteristic of `dropWhile` is that it stops as soon as the predicate fails. This is essentially the functional equivalent of a `break` statement in a `for` loop. The other interesting tid bit is the `lazy val`. It prevents hitting the cache twice, once to check if has a value and once to retreive it. It's a pretty sweet feature to have built into the language.

The problem is it took me longer than I expected to write the code and when I was done, I was concerned about the readability. So I decided to write the same code in Java. 

``` java
public class CompositeCache implements Cache {
    private List<Cache> caches = new ArrayList<Cache>();

    public CompositeCache add(Cache cache) {
        caches.add(cache);
        return this;
    }

    @Override
    public <T extends Serializable> Option<T> get(String key) {
        Option<T> value = Option.empty();

        List<Cache> misses = new ArrayList<Cache>();
        for (Cache cache : caches) {
            value = cache.get(key);
            if (value.isDefined()) {
                for (Cache miss : misses) {
                    miss.set(key, value.get());
                }
                break;
            }
            misses.add(cache);
        }

        return value;
    }

    @Override
    public void set(String key, Serializable value) {
        for (Cache cache : caches) {
            cache.set(key, value);
        }
    }
}
```

I wrote the code about as fast as I could type and I'm pretty sure you don't need me to break it down. I asked a coworker to take a look and he was of course able to instantly grok the Java code. He said something interesting though. He said maybe he was able to quickly understand the Java because he's been writing for the last 20 years.

That got me thinking. I've been writing imperative code forever so I subconciously dismiss all the boiler plate variable and loop hooha to quickly digest the logic. But really, my brain is doing gymnastics to translate the imperative implementation to declarative thoughts.

I don't know about you, but I don't think in terms of loops and state changes. I think in terms of actions.

Take a simple task like brushing your teeth and describe it to yourself. For me it goes something like this.

* Pick up toothbrush
* Put toothpaste on tootbrush
* Brush teeth
* Rinse toothbrush
* Put toothbrush back in holder

Or do you prefer

* Change amount of toothpaste on toothbrush from 0 to .1 oz.
* ...

So maybe function is easier to understand, but harder to write. Sometimes.