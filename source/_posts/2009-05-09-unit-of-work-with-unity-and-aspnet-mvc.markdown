---
layout: post
title: "Unit of Work with Unity and ASP.NET MVC"
date: 2009-05-09 21:42
comments: true
categories: [AOP, ASP.NET MVC, Domain Driven Design, NHibernate, Unity]
---

I was recently asked how I get the context of  “this” in the UoW relating to the current page request.

{% img plain /images/posts/howididit-thumb.jpg %}

Before I get into the guts, I would like to provide a little context. My application has 20+ databases scattered across 4 machines.

``` c#
IRepository<Customer> customerRepository; // customer database on server 1
IRepository<Package>  packageRepository; // customer database on server 1
IRepository<Contact>  contactRepository; // contact database on server 2
```

So, I might ask for a `Customer` object and a `Package` object and I want to get the same `ISession` for both and if I ask for the same Customer twice, I want to get the one from the 1st level cache (I’m using NHibernate). If I ask for a Contact object, I will get a different ISession. All opened sessions are managed by my UoW. So, when the page request is complete, I call `UoW.Commit` and all sessions are committed.

The “magic” if you will, happens in the global.asax. I was nosing around in Rhino.Commons for inspiration and adapted a technique I saw there. This is how it looks:

``` c#
public class GlobalApplication : HttpApplication
{
    private static IUnityContainer container;

    public GlobalApplication()
    {
        BeginRequest += new EventHandler(GlobalApplication_BeginRequest);
        EndRequest += new EventHandler(GlobalApplication_EndRequest);
    }

    protected void Application_Start(object sender, EventArgs e)
    {
        RegisterRoutes(RouteTable.Routes);

        container = new UnityContainer();
        container.AddNewExtension<PolicyInjectorContainerExtension>();
        container.AddNewExtension<HttpRequestLifetimeCoreContainerExtension>();
        container.AddNewExtension<WebMvcContainerExtension>();

        ControllerBuilder.Current.SetControllerFactory(new UnityControllerFactory(container));
    }

    void GlobalApplication_BeginRequest(object sender, EventArgs e)
    {
        var unitOfWork = container.Resolve<IUnitOfWork>();
        unitOfWork.Start();
    }

    void GlobalApplication_EndRequest(object sender, EventArgs e)
    {
        var unitOfWork = container.Resolve<IUnitOfWork>();
        unitOfWork.Commit();
    }
}
```

I register the UoW with an `HttpRequestLifetimeManager` so I get a new instance for each request.

``` c#
Container.RegisterType<IUnitOfWork<ISession>,
	NHibernateUnitOfWork>(new HttpRequestLifetimeManager());
```

My `NHibernateRepository` gets injected with the UoW for the current `HttpRequest` and when the request is complete, the `global.asax` commits the whole thing.

``` c#
public class NHibernateRepository<T> : IRepository<T>
{
    protected ISession session;

    public NHibernateRepository(IUnitOfWork<ISession> unitOfWork)
    {
        session = unitOfWork.GetContextFor<T>();
    }

    ...

    public virtual void Save(T obj)
    {
        session.Save(obj);
    }
}
```

Now, this is all the context of an ASP.NET MVC Controller, but I have a similar issue for other (non-web) services. In that context I am using AOP and decorating a particular method with `[UnitOfWork]`, which looks like:

``` c#
public class UnitOfWorkCallHander : ICallHandler
{
    private IUnitOfWork<ISession> unitOfWork;

    public UnitOfWorkCallHander(IUnitOfWork<ISession> unitOfWork)
    {
        this.unitOfWork = unitOfWork;
    }

    public int Order { get; set; }

    public IMethodReturn Invoke(IMethodInvocation input, GetNextHandlerDelegate getNext)
    {
        unitOfWork.Start();

        try
        {
            return getNext()(input, getNext);
        }
        finally
        {
            unitOfWork.Commit();
        }
    }
}
```

In that context I use a `PerThreadLifetimeManager` for the `NHibernateUnitOfWork`, and code ends up looking like:

``` c#
[UnitOfWork]
public void Process(Job job)
{
    ...
}
```

You know, there is really no reason why you couldn’t do the same thing in the MVC context. You could basically ditch the `global.asax` event hooha and just annotate the Controller task:

``` c#
[UnitOfWork]
public ActionResult ControllerTaskThatRequiresUoW()
{
    ...
}
```

It’s more explicit than using the `global.asax` technique and it would allow you to specify different UoW behavior on each controller task:

``` c#
[UnitOfWork(IsolationLevel.ReadCommitted]
public ActionResult SomeTaskThatShouldNotReadUncommitedData()
{
}

[UnitOfWork(IsolationLevel.ReadUncommitted)]
public ActionResult AnotherTaskWithDifferentRequirements()
{
}
```

I’d like to try this out next time I get back into ASP.NET MVC, the global.asax event technique felt a little too magical and I don’t think Uncle Bob would approve.