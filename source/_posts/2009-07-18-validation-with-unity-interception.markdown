---
layout: post
title: "Validation with Unity Interception"
date: 2009-07-18 18:20
comments: true
categories: [AOP, Composite Pattern, Unity, Validation]
---

{% img plain /images/posts/funny_sign_5.jpg %}

This is how I want things to look:

``` c#
public class UpdateUserRequest
{
    [AuthorizedUserPublisherRequired]
    public int UserId { get; set; }

    [Required("Name is required")]
    public string Name { get; set; }

    [ValidZipCodeRequired("Invalid zip code")]
    public string ZipCode { get; set; }
}

public interface IUserService
{
    void Update(UpdateUserRequest request);
}
```

There are validation frameworks out there that will do this, so what’s my beef? Well, first of all I want to inject the validators with dependencies to implement the juicier rules. And second, I’m treating validation as a core concern so I don’t want a dependency on a framework like Enterprise Library. I had several questions like:

1. How do I get dependencies into the validators?
2. How do I get `ValidZipCodeRequired` to fire with the value of the `ZipCode` property
3. How do I initiate this on `userService.Update()`?

Let’s take a look at building up the validators.

``` c#
public class ValidZipCodeRequiredAttribute : ValidatorAttribute
{
    public override IValidator CreateValidator(IValidatorFactory factory)
    {
        return new factory.Create<ValidZipCodeRequiredValidator>();
    }
}
```

This allows me to make a `UnityValidatorFactory` that can supply all my dependencies. Now how about this business of running the `ValidZipCodeRequiredValidator` with the value of the `ZipCode` property? For that I made a composite validator that loops through each property and runs the annotated validator against it’s property value.

``` c#
public class CompositeValidator<T> : IValidator<T>
{
    private IValidatorFactory factory;

    public CompositeValidator(IValidatorFactory factory)
    {
        this.factory = factory;
    }

    public IEnumerable<RuleException> Validate(T subject)
    {
        List<RuleException> exceptions = new List<RuleException>();

        foreach (PropertyInfo property in typeof(T).GetProperties())
        {
            foreach (var validator in GetValidatorsFor(property))
            {
                exceptions.AddRange(validator.Validate(property.GetValue(subject, null)));
            }
        }

        return exceptions;
    }

    private IEnumerable<IValidator> GetValidatorsFor(ICustomAttributeProvider provider)
    {
        foreach (ValidatorAttribute attribute
            in provider.GetCustomAttributes(typeof(ValidatorAttribute), true))
        {
            yield return attribute.CreateValidator(factory);
        }
    }
}
```

Now I just need to run the `CompositeValidator` on `userService.Update()`. For that I use Unity Interception:

``` c#
public class ValidationCallHandler : ICallHandler
{
    private IValidator validator;

    public ValidationCallHandler(IValidator validator)
    {
        this.validator = validator;
    }

    public IMethodReturn Invoke(IMethodInvocation input, GetNextHandlerDelegate getNext)
    {
        var ruleExceptions = ValidateEachArgument(input.Arguments);

        if (ruleExceptions.Count() > 0)
        {
            throw new ValidationException(ruleExceptions);
        }

        return getNext()(input, getNext);
    }

    private IEnumerable<RuleException> ValidateEachArgument(IParameterCollection arguments)
    {
        var ruleExceptions = new List<RuleException>();

        foreach (var arg in arguments)
        {
            ruleExceptions.AddRange(validator.Validate(arg));
        }

        return ruleExceptions;
    }

    public int Order { get; set; }
}
```

Phew, we’re almost done. The only thing left is to apply this validator in configuration so I can keep the Unity reference out of my core domain. That looks like this:

``` c#
container.AddNewExtension<Interception>()
    .Configure<Interception>()
    .SetInterceptorFor<IUserService>(new InterfaceInterceptor())
    .AddPolicy("UserServiceValidationPolicy")
    .AddCallHandler<ValidationCallHandler>()
    .AddMatchingRule<AlwaysApplyMatchingRule>();
```