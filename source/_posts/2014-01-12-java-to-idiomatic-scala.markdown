---
layout: post
title: "Java to idiomatic scala"
date: 2014-01-12 12:55
comments: true
categories:
published: false
---

I've got a simple caching interface and a couple of concrete implementations. One is a local cache using an LRU map and the other is a distributed cache backed by Redis.

``` scala
trait Cache {

  def get[A <: Serializable](key: String): Option[A]

  def set(key: String, value: Serializable)
}
```

Redis may be fast, but it's not as fast as a local cache. After all, data still has to be deserialized and sent across a network from a Redis server. But local memory is limited and not available to other machines in a cluster.

The best of both worlds can be had by putting the two caches together in a fall through scenario. The composite pattern should do nicely.

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

The `span` operator is basically the same as doing `dropWhile` and `takeWhile` at the same time and putting the result in a tuple. The important characteristic of `dropWhile` is that it stops as soon as the predicate fails. This is essentially the functional equivalent of a `break` statement in a `for` loop. The other interesting tid bit is the `lazy val`. It prevents hitting the cache twice, once to check if it has a value and once to retreive it. It's a pretty sweet feature to have built into the language.

It took me longer than I expected to write the code and when I was done, I was concerned about the readability. So I decided to write the same code in Java. 

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

I wrote the code about as fast as I could type and I'm pretty sure you don't need me to break it down.