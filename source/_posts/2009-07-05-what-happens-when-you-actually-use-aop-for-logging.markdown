---
layout: post
title: "What Happens When You Actually Use AOP for Logging?"
date: 2009-07-05 20:10
comments: true
categories: [AOP, Composite Pattern]
---

Everybody’s favorite example for AOP is logging. There are a few reasons for that. First, it’s a great problem to solve with AOP and chances are it’s something everyone can relate to. Well, what happens when you actually use it for logging?

I can tell you what happened to me last week. Things were going great until it came time to port our credit card charging program from the stone age to .NET. It occurred to me that our fancy AOP logging would unknowingly log decrypted credit cards!

 {% img plain /images/posts/credit_cards1.jpg %}

An obvious solution was to do a regex on log messages and mask credit card numbers – and that’s pretty much what we did, but I didn’t want to perform a regex on every message when I knew only a tiny percentage would contain sensitive data. The solution of course, was to fight fire with fire and use more AOP.

This is how I wanted it to look:

``` c#
public interface IPayPalGateway
{
    [MaskCredCardForLogging]
    string Submit(string request, string id);
}
```

This way I could specifically mark an interface that I knew would accept sensitive data and apply an appropriate filter to it. The log filtering is implemented as a decorator to a log4net-like ILogger interface.

``` c#
[TestFixture]
public class LoggerWithFilterFixture
{
    [Test]
    public void Should_apply_filter_to_message()
    {
        var logger = new InterceptingLogger();

        var loggerWithFilter = new LoggerWithFilter(logger, new AppendBangToMessage());

        loggerWithFilter.Debug("This is a message");

        Assert.AreEqual("This is a message!", logger.LastMessage);
    }
}

public class AppendBangToMessage : ILogFilter
{
    public string Filter(string message)
    {
        return message + "!";
    }
}
```

The decorator is then applied in the call handler by looking for custom attributes of type LogFilterAttribute like this:

``` c#
public class LoggerCallHandler : ICallHandler
{
    public IMethodReturn Invoke(IMethodInvocation input, GetNextHandlerDelegate getNext)
    {
        ILogger logger = GetLogger(input);

        if (logger.IsDebugEnabled)
        {
            logger.Debug(CreateLogMessage(input));
        }

        Stopwatch stopWatch = Stopwatch.StartNew();
        IMethodReturn result = getNext()(input, getNext);
        stopWatch.Stop();

        if (logger.IsDebugEnabled)
        {
            logger.Debug(CreateLogMessage(input, result, stopWatch));
        }

        return result; 
    }

    ...

    public ILogger GetLogger(IMethodInvocation input)
    {
        var logger = CreateLogger(input);

        var filter = CreateLogFilter(input);

        return new LoggerWithFilter(logger, filter);
    }

    private ILogFilter CreateLogFilter(IMethodInvocation input)
    {
        var composite = new CompositeLogFilter();

        foreach (LogFilterAttribute attribute
            in input.MethodBase.GetCustomAttributes(typeof(LogFilterAttribute), true))
        {
            composite.Add(attribute.CreateLogFilter());
        }

        return composite;
    }
}
```

I’ll spare you the actual credit card masking logic:

``` c#
public class MaskCreditCardForLoggingAttribute : LogFilterAttribute
{
    public override ILogFilter CreateLogFilter()
    {
        return new CreditCardMaskingLogFilter();
    }
}

public class CreditCardMaskingLogFilter : ILogFilter
{
    public string Filter(string message)
    {
        ...
    }
}
```