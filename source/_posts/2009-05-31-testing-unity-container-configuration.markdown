---
layout: post
title: "Testing Unity Container Configuration"
date: 2009-05-31 21:04
comments: true
categories: [Inversion of Control, Unity]
---

Before Inversion of Control, I would use configuration for a connection string, port number or other boring piece of information. Nowadays however, configuration can be a pretty hairy part of the application in its own right. Not necessarily the xml kind of configuration, just configuration. You know, the place where you use the “new “ keyword and essentially break all the principals you worked so hard to protect in the rest of your application. Uncle Bob referred to this as “the other side of the wall” in a [podcast with Scott Hanselman](http://hanselminutes.com/default.aspx?showID=163).

{% img plain /images/posts/afewgoodmenjacktruth-thumb.jpg %}

> Son, we live in a world that has walls, and those walls have to be guarded by men with guns. Who’s gonna do it? You? You, Lieutenant Weinberg? I have a greater responsibility than you can possibly fathom… – Colonel Nathan R. Jessep

In my case, I wanted to inject an encryption provider to do Rijndael encryption with a specific vector and all that. Since I already had another IEncryptionProvider registered with the container, I named the new one.

``` c#
Container.Configure<InjectedMembers>().ConfigureInjectionFor<RijndaelEncryptionProvider>(
    "MyEncryptionProvider",
    new InjectionConstructor(
        new ResolvedParameter<NoOpSaltStrategy>(),
        new RijndaelConfig { Hash = "SHA1", Vector = "notreal", Iterations = 2, Size = 256 }));

Container.Configure<InjectedMembers>().ConfigureInjectionFor<MyService>(
    new InjectionConstructor(
        new ResolvedParameter<IEncryptionProvider>("MyEncryptionProvider"),
        new ResolvedParameter<DataDropConfigSection>()));
```

What is a good way to make sure I’m wiring up Unity properly? I could just go black box and make sure my service can decrypt a string encrypted with a particular vector. That would be pretty BDDish and provide good insulation from volatile configuration code. In this application however, doing a legit integration test was a PITA for other reasons and I wanted some quick feedback on my Unity configuration.

A coworker and I decided to use a container extension to watch all the dependencies that were resolved and then tell us if we got the right one. The unit test looked like this:

``` c#
[TestFixture]
public class EncryptionProviderTests
{
    private IUnityContainer container;
    private WasResolvedContainerExtension wasResolvedContainerExtension;

    [SetUp]
    public void SetUp()
    {
        wasResolvedContainerExtension = new WasResolvedContainerExtension();

        container = new UnityContainer()
            .AddExtension(wasResolvedContainerExtension)
            .AddNewExtension<WebMvcContainerExtension>();
    }

    [TearDown]
    public void TearDown()
    {
        container.Dispose();
    }

    [Test]
    public void Should_inject_named_instance_of_encryption_provider()
    {
        var service = container.Resolve<MyService>();

        AssertNamedInstanceWasResolved<IEncryptionProvider>("MyEncryptionProvider");
    }

    private void AssertNamedInstanceWasResolved<T>(string name)
    {
        Assert.IsTrue(wasResolvedContainerExtension.WasResolved<T>(name));
    }
}
```

Granted it’s a little magical, but at least it reads well and a failure is pretty easy to track down from that point.  This is what the container extension looks like:

``` c#
public class WasResolvedContainerExtension : UnityContainerExtension
{
    private WasResolvedBuilderStrategy strategy;

    protected override void Initialize()
    {
        strategy = new WasResolvedBuilderStrategy();

        Context.Strategies.Add(strategy, UnityBuildStage.Creation);
    }

    public bool WasResolved<T>()
    {
        return WasResolved<T>(null);
    }

    public bool WasResolved<T>(string name)
    {
        return strategy.WasResolved<T>(name);
    }
}
```

The real work is done in the `BuilderStrategy` which looks like:

``` c#
public class WasResolvedBuilderStrategy : BuilderStrategy
{
    private IList<NamedTypeBuildKey> buildKeys = new List<NamedTypeBuildKey>();

    public override void PreBuildUp(IBuilderContext context)
    {
        buildKeys.Add((NamedTypeBuildKey)context.BuildKey);
    }

    public bool WasResolved<T>()
    {
        return WasResolved<T>(null);
    }

    public bool WasResolved<T>(string name)
    {
        var buildKey = (buildKeys.FirstOrDefault(k =>
        typeof(T).IsAssignableFrom(k.Type) && k.Name == name));

        return buildKey.Type != null;
    }
}
```

And there you have it, a pretty quick way to test Unity configuration.