---
layout: post
title: "Refactoring: Introduce Parameter Object"
date: 2010-02-15 06:49
comments: true
categories: [Introduce Parameter Object, NHibernate, Refactoring, Single Responsibility Principle]
---

Take a look at this function:

``` c#
public UserIndexViewModel GetIndexViewModel(int pageSize, int pageNumber) {
    var model = new UserIndexViewModel();

    var allUsers = Session.CreateCriteria<User>();

    model.Total = CriteriaTransformer
        .TransformToRowCount(allUsers)
        .UniqueResult<int>();

    model.Users = allUsers
        .SetFirstResult(pageSize * (pageNumber - 1) + 1)
        .SetMaxResults(pageSize)
        .AddOrder<User>(u => u.LastName, Order.Asc)
        .List<User>();

    return model;
}
```

There isn’t much going on here, but it could be better. I’m looking at the arguments `pageSize` and `pageNumber`. Notice how they both start with the prefix “page”. That’s a smell. The common prefix means the arguments are related, so let’s apply the _Introduce Parameter Object_ refactoring and see what happens.

``` c#
public class Page {
    private readonly int number;
    private readonly int size;

    public Page(int number, int size) {
        this.number = number;
        this.size = size;
    }

    public int Size {
        get { return size; }
    }

    public int Number {
        get { return number; }
    }

    public int FirstResult {
        get { return size * (number - 1) + 1; }
    }
}
```

What did we accomplish?

* Creating the parameter object gave us a place to put the logic of calculating the first result of the current page. This logic would otherwise be scattered around the application and even though it’s simple, it’s error prone.
* The logic of calculating the first result is testable.

``` c#
[Test]
public void Should_calculate_first_result_of_current_page() {
    Assert.That(new Page(1, 20).FirstResult, Is.EqualTo(1));
    Assert.That(new Page(2, 20).FirstResult, Is.EqualTo(21));
    Assert.That(new Page(3, 20).FirstResult, Is.EqualTo(41));
}
```

It’s not rocket science, but it still deserves a test. Our other option is spinning up Cassini and clicking around to make sure we got it right. That’s just silly. A little piece of me dies each time I press F5 to run a manual test.

Anyway, there is this thing that happens with single responsibility classes. Because their purpose is crisp, it draws out robustness. In this case, I’m thinking about what we should we do if we get zero or a negative value for page number. I’m a fail fast kind of guy, so I’d probably go with a guard clause.

``` c#
public Page(int number, int size) {
    GuardAgainstLessThanOne(number);
    ...
}
```

But you could silently correct it if that’s how you roll:

``` c#
public int Number {
    get { return Math.Max(1, number); }
}
```

The point is that this is an obvious scenario to consider when looking at such a special purpose object. All too often though, bits of logic are mixed together like 32 kindergarteners sitting next to each other on a rug during a lesson. The room is one fart or tickle away from erupting into total kaos. It’s roughly half teaching and half crowd control, only without the rubber bullets and pepper spray. Not surprisingly, overcrowded classrooms don’t work that well and over-ambitous functions don’t work that well either. Each little bit of logic needs some one on one with you. Give it a chance to be all it can be by pulling it aside and getting to know it a little better. You might be surprised.