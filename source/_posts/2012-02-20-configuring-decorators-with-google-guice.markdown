---
layout: post
title: "Configuring Decorators with Google Guice"
date: 2012-02-20 07:11
comments: true
categories: [Decorator Pattern, Google Guice, Inversion of Control, Open Closed Principle]
---

You have a few options and each have their trade-offs. The one I find least annoying requires using a binding annotation. Since I’m stuck using annotations with Guice anyway, using one more to facilitate a decorator seems like an acceptable concession. Before I go on though, I have to take a moment. My beef isn’t about verbose configuration or annotations, it’s that once again the documentation gets it all wrong and sends the impressionable reader down a misguided path. Let’s take a look at this excerpt from the Guice documentation for binding annotations:

``` java
public class RealBillingService implements BillingService {

  @Inject
  public RealBillingService(@PayPal CreditCardProcessor processor, ...) {
    ...
  }
```

This bit of innocuous code encourages the reader to squander the power of dependency inversion and reduce it to a clunky tool that makes unit testing a little bit easier. That sounds harsh, so let’s start by discussing what Guice is and the problem it solves.

Guice and the like are referred to as IoC containers. That’s Inversion of Control. It’s a pretty general principle and when applied to object oriented programming, it manifests itself in the form of a technique called _Dependency Inversion_. In terms of the `BillingService` example, it means the code depends on a `CreditCardProcessor` abstraction rather than new‘ing something specific like a `PayPalCreditCardProcessor`. Perhaps depends is an overloaded term here. With or without the new keyword, there is a dependency. In one case, a higher level module is responsible for deciding what implementation to use, and in the other case, the class itself decides that it’s using a `PayPalCreditCardProcessor`, period.

Writing all your classes to declare their dependencies leaves you with the tedious task of building up complex object graphs before you can actually use your object. This is where Guice comes in. It’s a tool to simplify the havoc wreaked by inverting your dependencies and it’s inevitable when guided by a few principles like DRY (Don’t Repeat Yourself). If you don’t believe me, go ahead a see for yourself. Write some truly SOLID code and you’ll end up writing an IoC container in the process.

So now that we’ve covered what Guice is and the problem it solves, we are ready to talk about what’s wrong with `@PayPal`. Specifying the concrete class you expect with an annotation is pretty much the same as just declaring the dependency explicitly. Sure, you get a few points for using an interface and injecting the object, but it’s really just going through the motions while entirely missing the point. It would be like the Karate Kid going into auto detailing after learning wax-on, wax-off.

Abstractions create seams in your code. It’s how you add new behavior as the application evolves and it’s the key to managing complexity. Since we’re looking at a billing example, let’s throw out a few requirements that could pop up. How about some protection against running the same transaction twice in a short time period. How about checking a blacklist of credit cards or customers. Or maybe you need a card number that always fails in a particular way so QA can test the sad path. Or maybe your company works with a few payment gateways and wants to choose the least cost option based on the charge amount or card type. In this little snippet of code, we’ve got 2 seams we can use to work in this behavior. We’ve got the `BillingService` and `CreditCardProcesor`.

Oh, wait a minute we’re declaring that we need the `PayPalCreditCardProcessor` with that annotation so now our code is rigid and we can’t inject additional behavior by wrapping it in a `DoubleChargeCreditCardProcessor`, open-closed style. That’s the ‘O’ in SOLID. So you’re probably thinking, why can’t you just change the annotation from `@PayPal` to `@DoubleCharge`? Let’s dive a little deeper into this example to find out:

``` java
public class DoubleChargeCreditCardProcessor implements CreditCardProcessor {

  @Inject
  public DoubleChargeCreditCardProcessor(CreditCardProcessor processor, ...) {
    ...
  }
```

I’m not going to rant about how extends is evil and that you’re better off with a decorator because [I’ve already done that](/inheritance-is-evil-the-story-of-the-epic-fail-of-dataannotationsmodelbinder/), and this article is about how to wire up a decorator with Guice. So the challenge here is how to configure the container to supply the correct credit card processor as the first dependency of our double charge processor which itself implements `CreditCardProcessor`. Looking at the Guice documentation, you would likely think the answer is to do this:

``` java
public class RealBillingService implements BillingService {

  @Inject
  public RealBillingService(@DoubleCharge CreditCardProcessor processor, ...) {
    ...
  }
```

``` java
public class DoubleChargeCreditCardProcessor implements CreditCardProcessor {

  @Inject
  public DoubleChargeCreditCardProcessor(@PayPal CreditCardProcessor processor, ...) {
    ...
  }
```

That’s wrong though. The `CreditCardProcessor` isn’t a _thing_, it’s a _seam_ and it’s where you put additional behavior like preventing duplicate charges in a short time period. If you look at the decorator, you’ll notice that it has nothing to do with PayPal. That’s because it’s a business rule and shouldn’t be mixed with integration code. Our business rule code and the PayPal integration code will likely live in different packages and the `CreditCardProcessor` abstraction could get assembled differently for any number of reasons. Maybe your application supports [multi-tenancy](/bolt-on-multi-tenancy-in-asp-net-mvc-with-unity-and-nhibernate-part-ii-commingled-data/) and each tenant can use a different payment gateway. We can’t reuse our double charge business rule if it’s hard-coded to wrap a PayPal processor, and that’s a problem.

While I don’t particularly like using annotations for this sort of thing, it’s not the root cause. As a mechanic, it works just fine and can help us accomplish our task. The problem is that the documentation is subtly wrong and encourages mis-use of this feature. The better way to use binding annotations and not undermine the point of injecting your dependencies is like so:

``` java
public class DoubleChargeCreditCardProcessor implements CreditCardProcessor {

  public static final String BASE = "DoubleChargeCreditCardProcessor.base";

  public DoubleChargeCreditCardProcessor(@Named(BASE) CreditCardProcessor processor, ...) {
    ...
  }
```

``` java
public class ConfigureCreditCardProcessor extends AbstractModule {

  @Override
  protected void configure() {

    bind(CreditCardProcessor.class).to(DoubleChargeCreditCardProcessor.class);

    bind(CreditCardProcessor.class)
      .annotatedWith(Names.named(DoubleChargeCreditCardProcessor.BASE))
      .to(PayPayCreditCardProcessor.class);
  }
}
```

The difference is subtle, but the devil is in the details. In this last example, the `DoubleChargeCreditCardProcessor` doesn’t know or care what implementation it’s decorating. It simply declares a name for it’s dependency so it can be referenced unambiguously in a configuration module. This moves the configuration logic to… well, configuration code. Now you can see that the code is once again flexible and you can easily imagine more sophisticated configuration logic that could consider tenant settings or environment variables in selecting the proper combination of credit card processors to assemble.