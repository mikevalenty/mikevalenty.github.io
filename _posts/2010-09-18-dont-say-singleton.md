---
title: "Don’t Say Singleton"
date: 2010-09-18 22:10:00 -0700
layout: post
categories:
---

I was in junior high around the time that Master of Puppets was released followed by *…And Justice For All*. Maybe it’s nostalgia, but those two albums were not only the best Metallica albums, they were perhaps the greatest albums of the entire genre. Something happened though and I don’t mean grunge.

<img src="/images/posts/grunge_thumb.jpg">

Metallica eventually succumbed to their own success and tried to crank out albums more accessible to wider audiences. This tension between music with something to say and music that appeals to the masses has been the bane of the industry since the invention of the radio.

If you don’t believe me, go ask 10 people what their favorite Metallica song is. You’re going to hear Enter Sandman unless you work at an auto shop or maybe guitar center in which case you’re probably not reading this article. You’re going to hear Enter Sandman because that song was engineered by producers and marketing types to be played on the same radio stations that host morning shows like Jeff and Jer, yet that song couldn’t be further from the band’s roots.

Likewise, if you ask 10 programmers if they use design patterns, you will undoubtedly hear a resounding yes citing the widely accessible singleton pattern. Why is this the case? It’s because the singleton most closely resembles something that looks familiar when writing procedural code and it’s the easiest pattern to insert into a pile of crap. The problem is that the singleton represents design patterns about as well as Enter Sandman captures the essence of why Metallica was a great band.

Pattern catalogs should probably look more like this:

**Design Patterns:**

*   Singleton*
*   Strategy
*   Decorator
*   …

*   *Not a design pattern*

Or at least come with a warning label.

> Warning: The Singleton is actually a global variable. Use this pattern as a last resort and be sure to hide the fact you are using it from the rest of your application and your manager.

So how do you know if you’re using the Singleton correctly? I’ll give you a hint, if you have any knowledge of the fact that the object you’re using is a singleton, then you’re not using it correctly. Let’s say you’re writing some caching logic since it’s one of those places where singletons pop up.

Let’s start with a simple interface for finding a user object:

```c#
public interface IUserRepository
{
    User FindById(int id);
}
```

You realize you keep doing the same lookup over and over and adding some caching would really perk up the application. Hooray, for the Singleton!

```c#
public class CachingUserRepository : SqlUserRepository
{
    public override User FindById(int id)
    {
        var user = MyCache.Instance.Get(id);

        if (user == null)
        {
            user = base.FindById(id);
            MyCache.Instance.Add(id, user);
        }

        return user;
    }
}
```

Well this probably works great in a console application or Windows service, but what if your code is running in a web application, or as a WCF service. What if your Windows service is distributed across multiple machines? Should you care how your application is deployed in your `CachingUserRepository`? No, you shouldn’t. That object has one responsibility, and it’s to manage collaboration with the cache. Let’s try this instead:

```c#
public class CachingUserRepository : SqlUserRepository
{
    private readonly ICacheProvider<User> cache;

    public CachingUserRepository(ICacheProvider<User> cache)
    {
        this.cache = cache;
    }

    public override User FindById(int id)
    {
        var user = cache.Get(id);

        if (user == null)
        {
            user = base.FindById(id);
            cache.Add(id, user);
        }

        return user;
    }
}
```

Maybe your cache should live for the lifetime of a single web request, or maybe a web session, or possibly a distributed cache is right for your scenario. The good news is that caching is one of those solved problems, like data access, so you can just pick your favorite library and plug it in to your application with a simple adapter:

```c#
HttpRequestCacheProvider
HttpSessionCacheProvider
HttpContextCacheProvider
MemcacheCacheProvider
EnterpriseLibraryCacheProvider
VelocityCacheProvider
```

Let’s say you still want to roll your own cache and you decide that singleton is right scope for you. That’s okay, just recognize that using a singleton is a configuration decision and should not leak out into your application. The only place that you should see that .Instance getter is where you bootstrap your application.
