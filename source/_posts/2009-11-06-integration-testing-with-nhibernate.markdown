---
layout: post
title: "Integration Testing with NHibernate"
date: 2009-11-06 21:59
comments: true
categories: [Decorator Pattern, Integration Testing, NHibernate]
---

While the first-level cache in NHibernate is great for production, it can be super annoying for integration tests. Basically I just want to make sure my mapping works and that when I save an entity it goes to the database.

``` c#
[TestFixture]
public class MyIntegrationFixture : IntegrationFixture {
    [Test]
    public void Can_add_elements() {
        var repository = Container.Resolve<IRepository<MyEntity>>();
 
        var entity = repository.FindById(1);
        entity.Add(new Element { Description = "The Description" });
        repository.Save(entity);
 
        Assert.That(repository.FindById(1).Elements.Count(), Is.EqualTo(1));
    }
}
```

The problem is the next time I call `FindById()` I get my object back from the first level cache `ISession`, so the Assert passes but looking at the SQL output by NHibernate I can see that no INSERT statement is happening. There are two things that NHibernate needs to do in order for the integration test to work as expected.

1. There needs to be an explicit transaction and it needs to be committed in order to actually send the INSERT statements to the database.
2. The entity needs to be evicted from the session in order to ensure that the next call to `FindById()` actually goes to the database.

It’s the job of the application to manage the scope of the unit of work. Typically this is per web request in a web app. In a test, it’s per test. This turns into a bunch of boiler-plate noise. Also, this business about evicting the session is an NHibernate specific thing and doesn’t belong in a test that is dealing with an IRepository abstraction.

Since I solve all my problems with a pipeline or decorator, I figured I’d decorate the `ISession` and commit and evict on the operations I care about:

``` c#
public class AutoCommitAndEvictSession : SessionDecorator {
    public AutoCommitAndEvictSession(ISession session)
        : base(session) {
    }
 
    public override object Save(object obj) {
        object result;
        using (var tx = Session.BeginTransaction()) {
            result = Session.Save(obj);
            tx.Commit();
        }
        Session.Evict(obj);
        return result;
    }
 
    public override void Update(object obj) {
        CommitAndEvict(base.Update, obj);
    }
 
    public override void SaveOrUpdate(object obj) {
        CommitAndEvict(base.SaveOrUpdate, obj);
    }
 
    public override void Delete(object obj) {
        CommitAndEvict(base.Delete, obj);
    }
 
    private void CommitAndEvict(Action<object> action, object entity) {
        using (var tx = Session.BeginTransaction()) {
            action.Invoke(entity);
            tx.Commit();
        }
        Session.Evict(entity);
    }
}
```

Then I just work it into a test fixture like this:

``` c#
[TestFixture]
public abstract class IntegrationFixture {
    protected IUnityContainer Container { get; private set; }
 
    [TestFixtureSetUp]
    public virtual void TestFixtureSetUp() {
        Container = new UnityContainer()
            .AddNewExtension<ConfigureNHibernate>();
    }
 
    [TestFixtureTearDown]
    public virtual void TestFixtureTearDown() {
        Container.Dispose();
    }
 
    [SetUp]
    public virtual void SetUp() {
        var session = Container.Resolve<ISessionFactory>().OpenSession();
        Container.RegisterInstance<ISession>(new AutoCommitAndEvictSession(session));
    }
 
    [TearDown]
    public virtual void TearDown() {
        Container.Resolve<ISession>().Dispose();
    }
}
```