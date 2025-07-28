---
title: "Console Application With IoC"
date: 2009-11-21 21:57:00 -0800
layout: post
categories:
- fluent interface
- inversion of control
---

I like to think of the `Main` method in a Console application like the `Global.asax`. Its single responsibility is to wire-up things to be run in a console context. Each environment has unique configuration requirements. For example, when using NHiberate in a web application the lifetime of the `ISession` is attached to the `HttpRequest`, however in the context of a console application it may be a singleton or “per job” in a windows service. I like this:

```c#
class Program
{
    static void Main(string[] args)
    {
        using (var container = new UnityContainer())
        {
            container
                .AddExtension(new ConfigureForConsole(args))
                .Resolve<MyApplication>()
                .Execute();
        }
    }
}
```
