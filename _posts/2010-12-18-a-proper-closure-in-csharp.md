---
title: "A Proper Closure in C#"
date: 2010-12-18 20:46:00 -0800
layout: post
categories:
---

You’ve seen code like this:

```c#
public class Order
{
    private ITaxCalculator taxCalculator;
    private decimal? tax;

    public Order(ITaxCalculator taxCalculator)
    {
        this.taxCalculator = taxCalculator;
    }

    public decimal CalculateTax()
    {
        if (!tax.HasValue)
        {
            tax = taxCalculator.Calculate(this);
        }

        return tax.Value;
    }

    ...
}
```

There are a few things to pick at. I’m looking at the member variable named `tax` that is null until you call `CalculateTax` for the first time. There isn’t anything to prevent the rest of the class from using the tax variable directly and possibly repeating the null check code in multiple places. I thought it would be fun to rewrite it using a closure. I don’t mean the kind of accidental closures we write with LINQ, but an honest to goodness proper closure.

```c#
public class Order
{
    public Order(ITaxCalculator taxCalculator)
    {
        CalculateTax = new Func<Func<decimal>>(() =>
        {
            decimal? tax = null;
            return () =>
            {
                if (!tax.HasValue)
                {
                    tax = taxCalculator.Calculate(this);
                }
                return tax.Value;
            };
        })();
    }

    public Func<decimal> CalculateTax { get; set; }

    ...
}
```

A closure is created around the tax variable so only the `CalculateTax` function has access to it. That’s pretty awesome. I wouldn’t have thought of using this technique before learning JavaScript all over again. It’s fun code to write, but it’s not going to get checked-in to source control. It’s basically a landmine for the next guy that has to make changes. The mental energy it takes to wrap your head around it is like listening to a joke with a really long setup and a lousy punch line. The solution is more complicated than the problem.

I still thought it was worth the mental exercise. Each language has its wheelhouse which makes a certain class of problems easy to solve. Admittedly this was a bit forced here, but opening your mind to other solutions may lead to a breakthrough solving a legitimately tough problem.
