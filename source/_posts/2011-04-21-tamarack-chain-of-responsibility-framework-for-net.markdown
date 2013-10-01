---
layout: post
title: "Tamarack: Chain of Responsibility Framework for .NET"
date: 2011-04-21 21:39
comments: true
categories: [Chain of Responsibility, Inversion of Control, Single Responsibility Principle]
---

The Chain of Responsibility is a key building block of extensible software.

> Avoid coupling the sender of a request to its receiver by giving more than one object a chance to handle the request. Chain the receiving objects and pass the request along the chain until an object handles it. – Gang of Four

Variations of this pattern are the basis for [Servlet Filters](http://www.oracle.com/technetwork/java/filters-137243.html), [IIS Modules and Handlers](http://learn.iis.net/page.aspx/366/developing-iis-70-modules-and-handlers-with-the-net-framework/) and several open source projects I’ve had the opportunity to work with including [Sync4J](https://www.forge.funambol.org/), [JAMES](http://james.apache.org/), [Log4Net](http://logging.apache.org/log4net/), [Unity](http://unity.codeplex.com/) and yes, even [Joomla](http://www.joomla.org/). It’s an essential tool in the OO toolbox and key in transforming rigid procedural code into a composable Domain Specific Language.

I’ve blogged about this pattern before so what’s new this time?

1. The next filter in the chain is provided via a delegate parameter rather than a property
2. The project is [hosted on github](https://github.com/mikevalenty/tamarack)
3. There is a [NuGet package](http://nuget.org/List/Search?packageType=Packages&searchCategory=All+Categories&searchTerm=tamarack) for it

{% img /images/posts/pkgmgr31.png %}

How does it work?
It’s pretty simple, there is just one interface to implement and it looks like this:

``` c#
public interface IFilter<T, TOut>
{
    TOut Execute(T context, Func<T, TOut> executeNext);
}
```

Basically, you get an input to operate on and a value to return. The `executeNext` parameter is a delegate for the next filter in the chain. The filters are composed together in a chain which is referred to as a Pipeline in the Tamarack framework. This structure is the essence of the Chain of Responsibility pattern and it facilitates some pretty cool things:

* Modify the input before the next filter gets it
* Modify the output of the next filter before returning
* Short circuit out of the chain by not calling the executeNext delegate

### Show me examples!

Consider a block of code to process a blog comment coming from a web-based rich text editor. There are probably several things you’ll want to do before letting the text into your database.

``` c#
public int Submit(Post post)
{
    var pipeline = new Pipeline<Post, int>()
        .Add(new CanoncalizeHtml())
        .Add(new StripMaliciousTags())
        .Add(new RemoveJavascript())
        .Add(new RewriteProfanity())
        .Add(new GuardAgainstDoublePost())
        .Finally(p => repository.Save(p));
 
    var newId = pipeline.Execute(post);
 
    return newId;
}
```

What about dependency injection for complex filters? Take a look at this user login pipeline. Notice the generic syntax for adding filters by type. Those filters are built-up using the supplied implementation of `System.IServiceProvider`. My favorite is `UnityServiceProvider`.

``` c#
public bool Login(string username, string password)
{
    var pipeline = new Pipeline<LoginContext, bool>(serviceProvider)
        .Add<WriteLoginAttemptToAuditLog>()
        .Add<LockoutOnConsecutiveFailures>()
        .Add<AuthenticateAgainstLocalStore>()
        .Add<AuthenticateAgainstLdap>()
        .Finally(c => false);
 
    return pipeline.Execute(new LoginContext(username, password));
}
```

Here’s another place you might see the chain of responsibility pattern. Calculating the spam score of a block of text:

``` c#
public double CalculateSpamScore(string text)
{
    var pipeline = new Pipeline<string, double>()
        .Add<SpamCopBlacklistFilter>()
        .Add<PerspcriptionDrugFilter>()
        .Add<PornographyFilter>()
        .Finally(score => 0);
 
    return pipeline.Execute(text);
}
```

Prefer convention over configuration? Try this instead:

``` c#
public double CalculateSpamScore(string text)
{
    var pipeline = new Pipeline<string, double>()
        .AddFiltersIn("Tamarack.Example.Pipeline.SpamScorer.Filters")
        .Finally(score => 0);
 
    return pipeline.Execute(text);
}
```

Let’s look at the `IFilter` interface in action. In the spam score calculator example, each filter looks for markers in the text and adds to the overall spam score by modifying the result of the next filter before returning.

``` c#
public class PerspcriptionDrugFilter : IFilter<string, double>
{
    public double Execute(string text, Func<string, double> executeNext)
    {
        var score = executeNext(text);
 
        if (text.Contains("viagra"))
            score += .25;
 
        return score;
    }
}
```

In the login example, we look for the user in our local user store and if it exists we’ll short-circuit the chain and authenticate the request. Otherwise we’ll let the request continue to the next filter which looks for the user in an LDAP repository.

``` c#
public class AuthenticateAgainstLocalStore : IFilter<LoginContext, bool>
{
    private readonly IUserRepository repository;
 
    public AuthenticateAgainstLocalStore(IUserRepository repository)
    {
        this.repository = repository;
    }
 
    public bool Execute(LoginContext context, Func<LoginContext, bool> executeNext)
    {
        var user = repository.FindByUsername(context.Username);
 
        if (user != null)
            return user.IsValid(context.Password);
 
        return executeNext(context);
    }
}
```

Why should I use it?

> It’s pretty much my favorite animal. It’s like a lion and a tiger mixed… bred for its skills in magic. – Napoleon Dynamite

It’s simple and mildly opinionated in effort to guide you and your code into The Pit of Success. It’s easy to write single responsibility classes and use inversion of control and composition and convention over configuration and lots of other goodness. Try it out. Tell a friend.