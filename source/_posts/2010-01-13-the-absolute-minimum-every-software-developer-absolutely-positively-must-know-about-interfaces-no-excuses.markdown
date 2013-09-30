---
layout: post
title: "The Absolute Minimum Every Software Developer Absolutely, Positively Must Know About Interfaces (No Excuses!)"
date: 2010-01-13 22:24
comments: true
categories: [Single Responsibility Principle]
---

<span style="font-size: 70%">
The title of this article was inspired by [The Absolute Minimum Every Software Developer Absolutely, Positively Must Know About Unicode and Character Sets (No Excuses!) ](http://www.joelonsoftware.com/articles/Unicode.html)
by Joel Spolsky
</span>

I recently saw this question on stackoverflow:

{% img /images/posts/quest.png %}

There were good points on both sides, but the majority of responses were from closet interface haters succumbing to the mob effect to say "me too". On the other side of the argument, many took cover behind regurgitated responses like interfaces are for mocking dependencies in unit tests and because people that write books say you should use them. So how is it that scores of professional software developers don’t understand or can’t communicate why and how to use the most fundamental of object-oriented language features? Perhaps there is a clue in the details of the original question:

> Is there some hidden benefit of using an interface when you have 1 version of a class and no immediate need to create an interface?

The scattered and ineffectual responses are an unfortunate casualty of the ubiquitous poor online examples found on the Internet. No doubt you’ve seen something like this before:

``` c#
public interface IVehicle {
    int NumberOfPassengers { get; }
}

public class Car : IVehicle {
    public int NumberOfPassengers { get { return 4; } }
}

public class Bus : IVehicle {
    public int NumberOfPassengers { get { return 20; } }
}
```

I can’t help but have visions of rich objects that model the physical world so well they put an end to wars and world hunger. You know, methods like `Bark()` on a `Dog` object. These examples aren’t wrong per se, but they usher the reader down a predictable and dead-end path. It’s like the horribly cliché [nine dots puzzle](http://en.wikipedia.org/wiki/Thinking_outside_the_box) in which the goal is to link all 9 dots using four straight lines or less, without lifting the pen.

Common sense suggests it can’t be done because we have subconsciously imposed a false restriction that the lines must be within the boundaries of the square (I won’t insult you with the hopelessly worn out catch phrase). The point I’m trying to make here is the examples out there unwittingly set us up for failure. In social science this is called [framing](http://en.wikipedia.org/wiki/Framing_(social_sciences). The human brain is an impressive problem solving machine, but we can be easily fooled by framing a problem in a way that our stereotypes and assumptions obscure the often obvious answer.

In the case of interfaces, the examples suggest the concrete implementations are mutually exclusive physical concepts like _Car_ and _Bus_, or perhaps `SqlUserStore` and `LdapUserStore`. It’s an easy concept to latch on to, and it’s the bane of good object oriented design.

To think of a user store as a thing is a mistake. `IUserStore` defines a seam in which responsibility crosses a boundary. In an MVC application, it’s the seam between the controller’s routing responsibility and the model’s business logic. If you only have one implementation of `IUserStore`, then you’d better take another look because that’s a smell.

The place to start looking is within your one and only implementation. Where is the logic about sending a confirmation email or checking if the username is available? Chances are it ended up in the controller leaving your model anemic or it ended up in your model violating the single responsibility principle. Either way it’s a big fail sandwich.

Again, this is because of the false assumption that implementations are mutually exclusive as in the _Car_ and _Bus_ example. In other words, if I have a _Car_, then I don’t have a _Bus_. This is the wrong way to think of objects. Of course they can be mutually exclusive, but they are far more powerful when they are composable. Consider these implementations of `IUserService`:

* `EmailConfirmationUserService`
* `ValidatingUserService`
* `SqlUserService`
* `CachingUserService`

Each concern is orthogonal to each other concern and they are combined together to create the overall behavior of the system. The interface facilitates composition which is the key to high-quality, robust software. So the real problem isn’t that interfaces are overused, it’s that interfaces are squandered.