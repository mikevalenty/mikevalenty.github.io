---
layout: post
title: "Moq Extension Methods for Unity"
date: 2009-10-31 06:21
comments: true
categories: [Mocking, Unity]
---

A few people have asked about the `RegisterMock` extension method used in another post. The usage looks like this:

``` c#
[Test]
public void Should_delete_removed_image()
{
    container.RegisterMock<IFileRepository>()
        .Setup(r => r.Delete(It.IsAny<IFile>()))
        .Verifiable();

    container.RegisterMock<IBusinessRepository>()
        .Setup(r => r.FindById(3))
        .Returns(CreateBusinessWith(new BusinessImage { ImageId = 4 }));

    var controller = container.Resolve<BusinessGalleryController>();
    controller.Delete(3, 4);

    container.VerifyMockFor<IFileRepository>();
}
```

It’s just a few helper extensions for using [Moq](http://code.google.com/p/moq/) with [Unity](http://www.codeplex.com/unity/) that cut down on the noise in tests. My friend Keith came up with it, I just happen to blog about it first. Here it is:

``` c#
public static class MoqExtensions
{
    public static Mock<T> RegisterMock<T>(this IUnityContainer container) where T : class
    {
        var mock = new Mock<T>();

        container.RegisterInstance<Mock<T>>(mock);
        container.RegisterInstance<T>(mock.Object);

        return mock;
    }

    /// <summary>
    /// Use this to add additional setups for a mock that is already registered
    /// </summary>
    public static Mock<T> ConfigureMockFor<T>(this IUnityContainer container) where T : class
    {
        return container.Resolve<Mock<T>>();
    }

    public static void VerifyMockFor<T>(this IUnityContainer container) where T : class
    {
        container.Resolve<Mock<T>>().VerifyAll();
    }
}
```