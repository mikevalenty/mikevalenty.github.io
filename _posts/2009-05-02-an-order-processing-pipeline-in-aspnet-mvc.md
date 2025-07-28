---
title: "An Order Processing Pipeline in ASP.NET MVC"
date: 2009-05-02 21:57:00 -0700
layout: post
categories:
- asp.net mvc
- domain driven design
- fluent interface
---

Lately, I can’t seem to shake the pipeline pattern. It keeps popping up in all my applications. It’s like when I think about buying a new car and then I start seeing them everywhere on road and think – have there always been that many out there?

So, here it is in action. Below is a controller from an ASP.NET MVC application. The method accepts an order in the form of xml for provisioning products in our system.

```c#
public ActionResult EnqueueOrders(string orderXml)
{
    var context = new OrderContext(orderXml);

    var pipeline = new FilterPipeline<OrderContext>()
        .Add(new RewriteLegacyUsernameFeature(productFinder))
        .Add(new ValidateOrderXml())
        .Add(new ValidateReferenceCode(customerFinder))
        .Add(new IgnoreOnException(new ValidateRemoteAccountUsernames(packageRepository)))
        .Add(new IgnoreOnException(new ValidateRemoteAccountUniqueEmail(packageRepository)))
        .Add(new EnqueueOrders(customerFacade)).When(new OrderHasNoErrors())
        .Execute(context);

    var xml = BuildResponseFromOrderContext(context);

    return new XmlResult(xml);
}
```

Most of it is just validation logic, but there are a few meatier pieces. Take a look a the first filter – RewriteLegacyUsernameFeature. If you’ve ever published an API that accepts xml, then you’ve probably wanted to change your schema 5 minutes later. The pipeline is great way to deal with transforming legacy xml on the way in.

Next is the implementation of the `FilterPipeline` which is essentially a chain of responsibility with a few convenience features.

```c#
public class FilterPipeline<T> : IFilter<T>
{
    private IFilterLink<T> head;

    public void Execute(T input)
    {
        head.Execute(input);
    }

    public IFilterConfiguration<T> Add(IFilterLink<T> link)
    {
        var specificationLink = new SpecificationFilterLink<T>(link);

        AppendToChain(specificationLink);

        return specificationLink;
    }

    public IFilterConfiguration<T> Add(IFilter<T> filter)
    {
        return Add(new FilterLinkAdapter<T>(filter));
    }

    public void AppendToChain(IFilterLink<T> link)
    {
        if (head == null)
        {
            head = link;
            return;
        }

        var successor = head;

        while (successor.Successor != null)
        {
            successor = successor.Successor;
        }

        successor.Successor = link;
    }
}
```

You might be scratching your head and wondering what this business is with both `IFilter<T>` and `IFilterLink<T>`. `IFilter<T>` is just a simpler version of `IFilterLink<T>` that doesn’t require the implementer to deal with calling the next link in the chain. The subject will always pass through, no short-circuiting, hence the pipeline.

My favorite part  is the `SpecificationFilterLink<T>` which is a decorator that uses a Specification to decide if the filter should be invoked. So you can do little readable snippets like:

```c#
pipeline.Add(new EnqueueOrders(customerFacade)).When(new OrderHasNoErrors());
```

Maybe this post will get it out of my system and I can move on to other solutions. It’s just so handy…
