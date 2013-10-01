---
layout: post
title: "Custom Model Binder in ASP.NET MVC"
date: 2009-12-31 14:59
comments: true
categories: [ASP.NET MVC, Command Query Separation, Decorator Pattern, Single Responsibility Principle]
---

The default model binder or `DataAnnotationsModelBinder` in ASP.NET MVC works well for simple data structures that basically map properties directly to form elements, but it quickly breaks down in a real world application. Take this simple scenario:

{% img plain /images/posts/brimley.png %}

The built-in model binder can handle the easy fields like `FirstName`, `LastName`, and `Email`, but it doesn’t know how to deal with the `Roles`. This is where the custom model binder comes in:

``` c#
public class UserModelBinder : IModelBinder {
    private readonly IModelBinder binder;

    public UserModelBinder(IModelBinder binder) {
        this.binder = binder;
    }

    public object BindModel(ControllerContext controllerContext, ...) {
        var user = (User)binder.BindModel(controllerContext, bindingContext);
        AddRoles(user, controllerContext);
        return user;
    }

    private static void AddRoles(User user, ControllerContext controllerContext) {
        foreach (var role in GetRoles(controllerContext)) {
            user.Add(role);
        }
    }

    private static IEnumerable<Role> GetRoles(ControllerContext controllerContext) {
        var roles = controllerContext.HttpContext.Request["roles"];
        if (roles == null) yield break;
        foreach (var roleId in roles.Split(',')) {
            yield return (Role)Convert.ToInt32(roleId);
        }
    }
}
```

And it gets wired-up in the `Global.asax` like this:

``` c#
var defaultBinder = new DataAnnotationsModelBinder();
ModelBinders.Binders.DefaultBinder = defaultBinder;
ModelBinders.Binders.Add(typeof(User), new UserModelBinder(defaultBinder));
```

Because `UserModelBinder` is a decorator, it preserves the base functionality (like validation) of the default binder and only adds the part about mapping the roles. This is good stuff because it keeps construction of the model out of the controller. That’s the Single-Responsibility Principle in action.

## Command Query Separation

The two functions `AddRoles`, and `GetRoles` are an example of command query separation (CQS). It would be easy to combine those two functions together and in fact that’s how it looked at first. So why bother separating them? Well it doesn’t really matter in this simple case, but it’s like how I tell my 3 year-old son to move his milk cup away from the edge of the table. I figure it’s a good habit that will prevent a few spills in the long run. The idea with CQS is there is a natural seam between finding objects and doing stuff with them. I buy that. I expect `GetRoles` to be more volatile than `AddRoles` or at least change for different reasons.