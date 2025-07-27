---
title: "Inheritance Is Evil: The Epic Fail of the DataAnnotationsModelBinder"
date: 2010-01-04 06:52:00 -0800
layout: post
categories:
- asp.net mvc
- decorator pattern
- single responsibility principle
---

The `DataAnnotationsModelBinder` is a pretty neat little class that enforces validation attributes on models like this one:

```c#
public class Product {
    public int Id { get; set; }

    [Required]
    public string Description { get; set; }

    [Required]
    [RegularExpression(@”^\$?\d+(\.(\d{2}))?$”)]
    public decimal UnitPrice { get; set; }
}
```

Just for the record, this is a [cool bit of code](http://aspnet.codeplex.com/Release/ProjectReleases.aspx?ReleaseId=24471) that was released to the community for backseat drivers like me to use and complain about. I’m really only bitter because I can’t use it. Let’s take a peek at the source code to see why:

```c#
/// <summary>
/// An implementation of IModelBinder which is designed to be a replacement
/// default model binder, adding the functionality of validation via the validation
/// attributes in System.ComponentModel.DataAnnotations from .NET 4.0.
/// </summary>
public class DataAnnotationsModelBinder : DefaultModelBinder {
    ...
}
```

The problem with this class is that it’s not just about validation. It’s more about binding with validation sprinkled in. The author even states this as a design goal with the comment “designed to be a replacement default model binder.” This is a violation of the *Single Responsibility Principle* and it’s the reason I can’t use it. The `DefaultModelBinder` does some basic reflection mapping of form elements to model properties. That’s cool, but what about a controller action that takes json, xml, or anything slightly more complicated than what the default binder can handle?

I’ll tell you what happens. You write a custom model binder which is pretty easy and happens to be a pretty rad extension point in the MVC framework. Consider this model binder that deserializes json:

```c#
public class JsonModelBinder<T> : IModelBinder {
    private string key;

    public JsonModelBinder(string requestKey) {
        this.key = requestKey;
    }

    public object BindModel(ControllerContext controllerContext, ...) {
        var json = controllerContext.HttpContext.Request[key];
        return new JsonSerializer().Deserialize<T>(json);
    }
}
```

That’s pretty cool, but now I’ve lost my validation! This my friend is the evil of inheritance and the epic fail of the `DataAnnotationsModelBinder`. So what’s wrong with inheritance, isn’t that what OOP is all about? The short answer is no (the long answer is also no.)

<img class="plain" src="/images/posts/petervenkmanghostbusters.jpg">

> Human sacrifice, dogs and cats living together… mass hysteria! — Dr. Peter Venkman

Inheritance is pretty enticing especially coming from procedural-land and it often looks deceptively elegant. I mean all I need to do is add this one bit of functionality to another other class, right? Well, one of the problems is that inheritance is probably the worst form of coupling you can have. Your base class breaks encapsulation by exposing implementation details to subclasses in the form of protected members. This makes your system rigid and fragile.

The more tragic flaw however is the new subclass brings with it all the baggage and opinion of the inheritance chain. Why does a model that enforces validation attributes care how the model is constructed? The answer is it shouldn’t, and to add insult to injury, the better design is actually easier to implement. Listen closely and you may hear the wise whispers of our forefathers…

<img class="plain" src="/images/posts/ward.jpg" title="Ward Cunningham" >
<img class="plain" src="/images/posts/martin.jpg" title="Marting Fowler" >
<img class="plain" src="/images/posts/kent.jpg" title="Kent Beck" >
<img class="plain" src="/images/posts/robert.jpg" title="Robert Martin" >

> Use the decorator pattern.

Is this really going to solve all my problems? The short answer is yes. Let’s take a look:

```c#
public class BetterDataAnnotationsModelBinder : IModelBinder {
    private IModelBinder binder;

    public BetterDataAnnotationsModelBinder(IModelBinder binder) {
        this.binder = binder;
    }

    public object BindModel(ControllerContext controllerContext, ...) {
        var model = binder.BindModel(controllerContext, bindingContext);
        AddValidationErrorsToModelState(model, controllerContext);
        return model;
    }

    ...
}
```

Now all this class does is enforce validation and we can use whatever model binder we want to construct our object. Let’s wire this up in the `Global.asax` real quick:

```c#
var productModelBinder = new JsonModelBinder<Product>("product");

ModelBinders.Binders.Add(
    typeof(Product),
    new BetterDataAnnotationsModelBinder(productModelBinder));
```

This is cool for the following reasons:

1.  We are not coupled to the `DefaultModelBinder`. This means we don’t know or care about the structure and implementation details of it so it’s free to change in the next version of ASP.NET MVC.
2.  The code is simpler because it is only about validation.
3.  The behavior is composable. This means we can add validation to any model binder (like our json model binder) and bolt on more responsibilities with a chain of decorators. How about converting DateTime fields to UTC or striping formatting characters from phone numbers?
4.  It’s easy to test.
5.  You can substitute a decorator for a subclass in most cases and in most cases it’s the right choice. You’re better off making the decorator your default choice and only settle for inheritance when you really need it.

<img class="plain" src="/images/posts/kip.jpg">

> …don’t be jealous that I’ve been chatting online with babes all day. Besides, we both know that I’m training to be a cage fighter. — Kip
