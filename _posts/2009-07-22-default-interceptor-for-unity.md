---
title: "Default Interceptor for Unity"
date: 2009-07-22 18:17:00 -0700
layout: post
categories:
- aop
- unity
---

> Default: the two sweetest words in the English language – Homer Simpson

Unity interception requires you to specify what kind of interceptor to use for each type you want to intercept. This can be a little painful, so I decided to take a stab at a container extension that will apply a default interceptor with optional matching rules.

This is how I want it to look:

```c#
container
    .AddNewExtension<InterceptOnRegister>()
    .Configure<InterceptOnRegister>()
    .SetDefaultInterceptor<TransparentProxyInterceptor>()
    .AddMatchingRule(new AssemblyNameStartsWith("MyCompany"));
```

The basic idea here is to subscribe to the register events and configure interception if it meets our criteria. By default it will match everything and apply `TransparentProxyInterceptor`, but as you can see from the snippet above, you can explicitly supply another interceptor and matching rules.

Here it is:

```c#
public class InterceptOnRegister : UnityContainerExtension {
    private IInstanceInterceptor interceptor;
    private readonly IList<IInterceptRule> rules;

    public InterceptOnRegister() {
        interceptor = new TransparentProxyInterceptor();
        rules = new List<IInterceptRule> {
            new NotUnityInterceptionAssembly(),
            new DoesNotHaveGenericMethods() };
    }

    public InterceptOnRegister AddMatchingRule(IInterceptRule rule) {
        rules.Add(rule);
        return this;
    }

    public InterceptOnRegister AddNewMatchingRule<T>() where T : IInterceptRule, new() {
        rules.Add(new T());
        return this;
    }

    public InterceptOnRegister SetDefaultInterceptor<T>() where T : IInstanceInterceptor, new() {
        interceptor = new T();
        return this;
    }

    protected override void Initialize() {
        AddInterceptionExtensionIfNotExists();

        Context.Registering +=
            (sender, e) => SetInterceptorFor(e.TypeFrom, e.TypeTo ?? e.TypeFrom);
    }

    private void AddInterceptionExtensionIfNotExists() {
        if (Container.Configure<Interception>() == null) {
            Container.AddNewExtension<Interception>();
        }
    }

    private void SetInterceptorFor(Type typeToIntercept, Type typeOfInstance) {
        if (!AllMatchingRulesApply(typeToIntercept, typeOfInstance)) {
            return;
        }

        if (interceptor.CanIntercept(typeOfInstance)) {
            Container
                .Configure<Interception>()
                .SetDefaultInterceptorFor(typeOfInstance, interceptor);
        } else if (interceptor.CanIntercept(typeToIntercept)) {
            Container
                .Configure<Interception>()
                .SetDefaultInterceptorFor(typeToIntercept, interceptor);
        }
    }

    private bool AllMatchingRulesApply(Type typeToIntercept, Type typeOfInstance) {
        foreach (var rule in rules) {
            if (!rule.Matches(typeToIntercept, typeOfInstance)) {
                return false;
            }
        }

        return true;
    }
}
```

And you’ll need this stuff too:

```c#
public interface IInterceptRule {
    bool Matches(Type typeToIntercept, Type typeOfInstance);
}

public class NotUnityInterceptionAssembly : IInterceptRule {
    public bool Matches(Type typeToIntercept, Type typeOfInstance) {
        return !typeToIntercept.Assembly.Equals(typeof(Interception).Assembly);
    }
}

public class DoesNotHaveGenericMethods : IInterceptRule {
    public bool Matches(Type typeToIntercept, Type typeOfInstance) {
        return typeOfInstance.GetMethods().Count(m => m.IsGenericMethod) == 0;
    }
}

public class AssemblyNameStartsWith : IInterceptRule {
    private string match;

    public AssemblyNameStartsWith(string match) {
        this.match = match;
    }

    public bool Matches(Type typeToIntercept, Type typeOfInstance) {
        return typeOfInstance.Assembly.FullName.StartsWith(match);
    }
}
```
