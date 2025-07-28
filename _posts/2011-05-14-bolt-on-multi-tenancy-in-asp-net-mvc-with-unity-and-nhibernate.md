---
title: "Bolt-on Multi-Tenancy in ASP.NET MVC With Unity and NHibernate"
date: 2011-05-14 13:09:00 -0700
layout: post
categories:
- asp.net mvc
- inversion of control
- nhibernate
- open closed principle
- unity
---

### The Mission:

Build a web application as though it’s for a single customer (tenant) and add multi-tenancy as a bolt-on feature by writing only new code. There are flavors of multi-tenancy, in this case I want each tenant to have its own database but I want all tenants to share the same web application and figure out who’s who by looking at the host header on the http request.

### The Plan:

To pull this off, we’re going to have to rely on our SOLID design principles, especially Single Responsibility and Dependency Inversion. We’ll get some help from these frameworks:

*   [ASP.NET MVC](http://www.asp.net/mvc)
*   [Unity](http://unity.codeplex.com/)
*   [NHibernate](http://nhforge.org/Default.aspx)

### Game on:

Let’s take a look at a controller that uses NHibernate to talk to the database. I’m not going to get into whether you should talk directly to NHibernate from the controller or go through a service layer or repository because it doesn’t affect how we’re going to add multi-tenancy. The important thing here is that the `ISession` is injected into the controller, and we aren’t using the service locator pattern to request the `ISession` from a singleton.

```c#
public class UserController : Controller
{
    private readonly ISession session;

    public UserController(ISession session)
    {
        this.session = session;
    }

    public ActionResult Edit(int id)
    {
        var user = session.Load<User>(id);

        return View(user);
    }
}
```

Alright, now it’s time to write some new code and make our web application connect to the correct database based on the host header in the http request. First, we’ll need a database to store a list of tenants along with the connection string for that tenant’s database. Here’s my entity:

```c#
public class Tenant
{
    public virtual string Name { get; set; }

    public virtual string Host { get; set; }

    public virtual string ConnectionString { get; set; }
}
```

I’ll use the repository pattern here so there is a crisp consumer of the `ISession` that connects to the lookup database rather than one of the tenant shards. This will be important later when we go to configure Unity.

```c#
public class NHibernateTenantRepository : ITenantRepository
{
    private readonly ISession session;
    private readonly HttpContextBase context;

    public NHibernateTenantRepository(ISession session, HttpContextBase context)
    {
        this.session = session;
        this.context = context;
    }

    public Tenant Current
    {
        get
        {
            var host = context.Request.Headers["Host"];
            return FindByHost(host);
        }
    }

    public Tenant FindByHost(string host)
    {
        return session
            .Query<Tenant>()
            .SingleOrDefault(t => t.Host == host);
    }
}
```

So now we need a dedicated `ISessionFactory` for the lookup database and make sure that our `NHibernateTenantRepository` gets the right `ISession`. It’s not too bad, we just need to name them in the container so we can refer to them explicitly.

```c#
Container
    .RegisterType<ISessionFactory>(
        "tenant_session_factory",
        new ContainerControlledLifetimeManager(),
        new InjectionFactory(c =>
            new NHibernate.Cfg.Configuration().Configure().BuildSessionFactory())
    );

Container
    .RegisterType<ISession>(
        "tenant_session",
        new PerRequestLifetimeManager(),
        new InjectionFactory(c =>
            c.Resolve<ISessionFactory>("tenant_session_factory").OpenSession())
    );

Container
    .RegisterType<ITenantRepository, NHibernateTenantRepository>()
    .RegisterType<NHibernateTenantRepository>(
        new InjectionConstructor(
            new ResolvedParameter<ISession>("tenant_session"),
            new ResolvedParameter<HttpContextBase>()
        )
    );
```

Hopefully that’s about what you were expecting since it’s not really the interesting part. The more interesting part is configuring the `ISession` that gets injected into the `UserController` to connect to a different database based on the host header in the http request. The Unity feature we’re going to leverage for this is the `LifetimeManager`. This is an often overlooked feature of IoC containers.

```c#
Container
    .RegisterType<ISessionFactory>(
        new PerHostLifetimeManager(() => new HttpContextWrapper(HttpContext.Current)),
        new InjectionFactory(c =>
        {
            var connString = c
                .Resolve<ITenantRepository>()
                .Current
                .ConnectionString;

            return new NHibernate.Cfg.Configuration()
                .Configure()
                .SetProperty(NHibernate.Cfg.Environment.ConnectionString, connString)
                .BuildSessionFactory();
        }));

Container
    .RegisterType<ISession>(
        new PerRequestLifetimeManager(),
        new InjectionFactory(c => c.Resolve<ISessionFactory>().OpenSession())
    );
```

Here we’re using a custom `PerHostLifetimeManager`. This tells Unity to maintain a session factory per host. When Unity runs across a host it doesn’t have a session factory for, it will run the `InjectionFactory` block to create one using the connection string associated with that tenant.

Since multiple simultaneous requests will be trying to get and set values with the same key, we need to make sure our `PerHostLifetimeManager` is thread safe. That’s pretty easy since Unity comes with a `SynchronizedLifetimeManager` base class that takes care of the fact that `Dictionary` [isn’t thread safe](http://stackoverflow.com/questions/157933/whats-the-best-way-of-implementing-a-thread-safe-dictionary-in-net).

```c#
public class PerHostLifetimeManager : SynchronizedLifetimeManager
{
    private readonly Func<HttpContextBase> context;
    private readonly IDictionary<string, object> store;

    public PerHostLifetimeManager(Func<HttpContextBase> context)
    {
        this.context = context;
        store = new Dictionary<string, object>();
    }

    protected override object SynchronizedGetValue()
    {
        var host = GetHost();

        if (!store.ContainsKey(host))
            return null;

        return store[host];
    }

    protected override void SynchronizedSetValue(object newValue)
    {
        store[GetHost()] = newValue;
    }

    private string GetHost()
    {
        return context().Request.Headers["Host"];
    }
}
```

So what did we accomplish? Well we didn’t touch any of our existing application code. We just wrote new code and through configuration we added multi-tenancy! That’s pretty cool, but was it worth it? Well, the goal in itself isn’t super important, but this exercise can certainly highlight areas of your codebase where you might be violating the single responsibility principle or leaking too many infrastructure concepts into your application logic.
