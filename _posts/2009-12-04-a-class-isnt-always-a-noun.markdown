---
title: "A Class Isn’t Always a Noun"
date: 2009-12-04 12:45:00 -0800
layout: post
categories:
- chain of responsibility
- domain driven design
---

There is a convention programmers go by that says object names should be nouns and methods names should start with a verb. That’s crap and I’ll tell you why. First off, it’s an old rule kind of like Hungarian notation. Okay that was low. To be fair, I would probably recommend this rule to a college student.

However, if you’re doing this as your full-time job and you’re neck deep in a complex business domain, then you are seriously selling yourself short. When working in a team environment you’ve got to use every means possible to communicate what you were thinking and what hard-earned knowledge you gained along the way. Some poor sucker is going to open your project at some point and your code should be screaming important concepts right from the class list.

<img src="/images/posts/yext1.png">

I don’t expect you to understand how the app works, but you should at least get a feel right away for the things that are important. Knowing it’s a console app, you could find the entry point and quickly navigate to the meat and potatoes of the application.

```c#
public class SyncListingJob : MarshalByRefObject, IJob
{
    private IServiceProvider locator;
    private YextListing listing;

    public SyncListingJob(IServiceProvider locator)
    {
        this.locator = locator;
    }

    public void Init(YextListing listing)
    {
        this.listing = listing;
    }

    [UnitOfWork]
    public void Execute()
    {
        GuardAgainstNotInitialized();
        SyncListing();
    }

    private void SyncListing()
    {
        new LocatorChain<Yextlisting>(locator)
            .AddNew<RevertDiscontinuedListing>()
            .AddNew<RefreshLinkedBusiness>()
            .AddNew<CreateLinkToExistingBusiness>()
            .AddNew<CreateNewBusinessAndLink>()
            .Process(listing);
    }

    private void GuardAgainstNotInitialized()
    {
        if (listing == null) throw new Exception("Job not initialized!");
    }
}
```

* The `LocatorChain<T>` is an implementation of the chain of responsibility pattern.

I do everything I can to push out all the noise and bring the behavior to the surface. Sometimes the right way to name a class is after a feature or behavior. I want my code to read like a DSL and making everything a noun just doesn’t cut it. The drivers for me are context and readability.
