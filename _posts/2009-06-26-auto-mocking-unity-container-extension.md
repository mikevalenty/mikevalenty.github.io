---
title: "Auto-Mocking Unity Container Extension"
date: 2009-06-26 20:13:00 -0700
layout: post
categories:
- tdd
- unity
---

Using a container for unit testing is a good way to insulate your tests from changes in object construction. Typically my first test will be something pretty boring just to get the process started. A few tests later, the real behavior comes out along with new dependencies. Rather than having to go back and add the dependencies to all the tests I’ve already written, I like to use a container to build up the object I’m testing.

To streamline this process, I thought it would be handy to use a container extension to auto generate mocks for any required interface that wasn’t explicitly registered. The best way I can explain this is with code, so here it is:

```c#
[SetUp]
public void SetUp()
{
    container = new UnityContainer()
        .AddNewExtension<AutoMockingContainerExtension>();
}

[Test]
public void Should_be_really_easy_to_test()
{
    container
        .RegisterMock<IDependencyThatNeedsExplicitMocking>()
        .Setup(d => d.MyMethod(It.IsAny<int>()))
        .Returns("I want to specify the return value");

    var service = container.Resolve<ServiceWithThreeDependencies>();
    var result = service.DoSomething();

    Assert.That(result, Is.EqualTo("I didn't have to mock the other 2 dependencies!"));
}
```

It really reduces the noise level in tests and lets you focus on the interesting parts of your application. Here’s the code for the container extension:

<img src="/images/posts/luigi1.jpg">

```c#
public class AutoMockingContainerExtension : UnityContainerExtension
{
    protected override void Initialize()
    {
        var strategy = new AutoMockingBuilderStrategy(Container);

        Context.Strategies.Add(strategy, UnityBuildStage.PreCreation);
    }

    class AutoMockingBuilderStrategy : BuilderStrategy
    {
        private readonly IUnityContainer container;

        public AutoMockingBuilderStrategy(IUnityContainer container)
        {
            this.container = container;
        }

        public override void PreBuildUp(IBuilderContext context)
        {
            var key = context.OriginalBuildKey;

            if (key.Type.IsInterface && !container.IsRegistered(key.Type))
            {
                context.Existing = CreateDynamicMock(key.Type);
            }
        }

        private static object CreateDynamicMock(Type type)
        {
            var genericMockType = typeof(Mock<>).MakeGenericType(type);
            var mock = (Mock)Activator.CreateInstance(genericMockType);
            return mock.Object;
        }
    }
}
```

In case you’re wondering about `container.RegisterMock<>`, it’s just extension method that you can read about [here](/moq-extension-methods-for-unity/).
