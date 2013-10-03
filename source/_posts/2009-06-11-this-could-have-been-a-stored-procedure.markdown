---
layout: post
title: "This Could Have Been a Stored Procedure"
date: 2009-06-11 20:56
comments: true
categories: [Domain Driven Design, Specification Pattern]
---

{% img plain right /images/posts/domaindrivendesignbookcover.jpg %}

I read [Domain Driven Design](http://www.amazon.com/Domain-Driven-Design-Tackling-Complexity-Software/dp/0321125215) about a year and a half ago and when I got to the part about the specification pattern, I thought it was really cool and I couldn’t wait to try it out. Maybe the pattern gods were listening or maybe I was unknowingly using [the secret](http://www.thesecret.tv/). Either way, it was pretty much the next morning that I went to work and had the perfect project dropped in my lap.

Our phone system routes live phone calls to doctor’s offices and the project was to bill the doctor if the call met the following criteria:

1. Caller is a new patient (pressed “1” instead of “2” in response to a voice prompt)
2. A live call was connected for more than 20 seconds or message longer than 10 seconds was recorded.
3. The doctor has not already been billed for this caller.
4. The call is not from a known list of test numbers.

The application subscribes to an event stream of new call records and runs them through this composite specification to determine if the call is billable.

``` c#
public bool IsBillable(CallRecord call)
{
    ISpecification<CallRecord> billableCallSpecification = new NewPatient()
        .And(new MinLengthLiveCall(liveCallSeconds).Or(new MinLengthMessage(messageSeconds))
        .And(new RepeatCall(referralFinder).Not())
        .And(new TestCall(testNumbers).Not()));

    return billableCallSpecification.IsSatisfiedBy(call);
}
```

I have to admit that I was already an hour into Query Analyzer blasting through another monster stored procedure when I caught myself. Not just the criteria logic either, I mean bypassing all the event stream hooha and basically just writing the whole enchilada as one gnarly stored procedure that would run as a job. That was a close one!