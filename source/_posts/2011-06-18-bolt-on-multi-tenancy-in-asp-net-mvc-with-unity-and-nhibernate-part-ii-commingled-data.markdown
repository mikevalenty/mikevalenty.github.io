---
layout: post
title: "Bolt-on Multi-Tenancy in ASP.NET MVC with Unity and NHibernate: Part II – Commingled Data"
date: 2011-06-18 11:08
comments: true
categories: [ASP.NET MVC, Inversion of Control, NHibernate, Open Closed Principle, Unity]
---

Last time I went over going from separate web applications per tenant to a shared web application for all tenants, but each tenant still had its own database. Now we’re going to take the next step and let multiple tenants share the same database. After we add `tenant_id` to most of the tables in our database we’ll need the application to take care of a few things. First, we need to apply a where clause to all queries to ensure that each tenant sees only their data. This is pretty painless with NHibernate, we just have to define a parameterized filter:

``` xml
<hibernate-mapping xmlns="urn:nhibernate-mapping-2.2">
 
  <filter-def name="tenant">
    <filter-param name="id" type="System.Int32" />
  </filter-def>
 
</hibernate-mapping>
```

And then apply it to each entity:

``` xml
<class name="User" table="[user]">
 
  <id name="Id" column="user_id">
    <generator class="identity" />
  </id>
 
  <property name="Username" />
  <property name="Email" />
 
  <filter name="tenant" condition="tenant_id = :id" />
 
</class>
```

The last step is to set the value of the filter at runtime. This is done on the `ISession` like this:

``` c#
Container
    .RegisterType<ISession>(
        new PerRequestLifetimeManager(),
        new InjectionFactory(c =>
        {
            var session = c.Resolve<ISessionFactory>().OpenSession();
            session.EnableFilter("tenant").SetParameter("id", c.Resolve<Tenant>().Id);
            return session;
        })
    );
```

The current tenant comes from `c.Resolve<Tenant>()`. In order for that to work, you have to tell Unity how to find the current tenant. In ASP.NET MVC, we can look at the host header on the request and find our tenant that way. We could just as easily use another strategy though. Maybe if this were a WCF service, we could use an authentication header to establish the current tenant context. You could build out some interfaces and strategies around establishing the current tenant context, however for this article I’ll just bang it out.

``` c#
Container
    .RegisterType<Tenant>(new InjectionFactory(c =>
    {
        var repository = c.Resolve<ITenantRepository>();
 
        var context = c.Resolve<HttpContextBase>();
 
        var host = context.Request.Headers["Host"] ?? context.Request.Url.Host;
 
        return repository.FindByHost(host);
    }));

```

Second, we have to set the `tenant_id` when new entities are saved. This is a bit more complicated with NHibernate and requires a bit of a concession in that we have to add a field to the entity in order for NHibernate to know how to persist the value. I’m using a private nullable int for this.

``` c#
public class User
{
    private int? tenantId;
 
    public virtual int Id { get; set; }
 
    public virtual string Username { get; set; }
 
    public virtual string Email { get; set; }
}
```

It’s private because I don’t want the business logic to deal with it and it’s nullable because my tenant table is in a separate database which means I can’t lean on the data model to enforce referential integrity. That’s a problem because the default value for an integer is zero which could be happily saved by the database. By making it nullable I can be sure the database will blow up if the `tenant_id` is not set.

So, back to the issue at hand. The `tenant_id` needs to be set when the entity is saved. For this, I’m using an interceptor and setting the value in the `OnSave` method:

``` c#
public class MultiTenantInterceptor : EmptyInterceptor
{
    private readonly Func<Tenant> tenant;
 
    public MultiTenantInterceptor(Func<Tenant> tenant)
    {
        this.tenant = tenant;
    }
 
    public override bool OnSave(object entity... object[] state, string[] propertyNames...)
    {
        var index = Array.IndexOf(propertyNames, "tenantId");
 
        if (index == -1)
            return false;
 
        var tenantId = tenant().Id;
 
        state[index] = tenantId;
 
        entity
            .GetType()
            .GetField("tenantId", BindingFlags.Instance | BindingFlags.NonPublic)
            .SetValue(entity, tenantId);
 
        return false;
    }
}
```

This `IInterceptor` mechanism is a little wonky. If you change any data, you have to do it in both the entity instance and the state array that NHibernate uses to hydrate entities. It’s not a big deal, it’s just one of those things you have to accept like the fact that Apple and Google are tracking your every move via your smart phone. Oh, and the interceptor gets wired up like this:

``` c#
Container
    .RegisterType<ISessionFactory>(
        new ContainerControlledLifetimeManager(),
        new InjectionFactory(c =>
        {
            return new NHibernate.Cfg.Configuration()
                .Configure()
                .SetInterceptor(new MultiTenantInterceptor(() => c.Resolve<Tenant>()))
                .BuildSessionFactory();
        })
    );
```

We’re almost done. There is one more case that needs to be handled. When NHibernate loads an entity by its primary key, it doesn’t run through the query engine which means the tenant filter isn’t applied. Fortunately, we can take care of this in the interceptor:

``` c#
public class MultiTenantInterceptor : EmptyInterceptor
{
    ...
 
    public override bool OnLoad(object entity... object[] state, string[] propertyNames...)
    {
        var index = Array.IndexOf(propertyNames, "tenantId");
 
        if (index == -1)
            return false;
 
        var entityTenantId = Convert.ToInt32(state[index]);
 
        var currentTenantId = tenant().Id;
 
        if (entityTenantId != currentTenantId)
        {
            throw new AuthorizationException("Permission denied to {0}", entity);
        }
 
        return false;
    }
}
```

That’s it. Have fun and happy commingling.