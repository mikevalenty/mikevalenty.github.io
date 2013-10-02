---
layout: post
title: "Definition of Enterprise Software"
date: 2009-12-06 12:41
comments: true
categories: [AOP, Enterprise Library, Inversion of Control]
---

The difference between Enterprise software and regular software is like the difference between a start-up and a public company. Enterprise software has lots of bureaucracy and red tape (indirection and configuration) to support transparency and top-down policy. You pay a price up front to create all these layers and well-defined communication channels between departments, but once the whole thing is up and running, it’s powerful.

Big companies can throw time and money at problems to produce activity diagrams, advisory boards and flow charts. Sure there’s bloat, but you need that kind of oversight to enforce corporate policy. Similarly, enterprise software can extravagantly use cpu cycles, disk space and memory, but this is a fair price to pay when complexity is king. Enterprise software must be concerned with things like service level agreements and government regulations. So how do you do that?

You do that by dissecting your application into composable units of behavior. This is the essence of the single-responsibility-principle and while the concept is deceptively simple, it can have a profound effect on your software. Yes, the end result is lots of little classes, but it’s much more than that. These classes capture individual business concepts and overtime your software becomes a domain-specific-language (DSL). I don’t mean the kind of DSL that a business person would use directly, but more like xpath and regular expressions except your DSL is focused on your business domain.

Over time, software becomes an asset or a liability. The difference is your ability to respond to the needs of the business and your toolbox for that is your DSL or lack thereof. You need to compose, decompose, rearrange and built-out bits of behavior in ways impossible to predict. Single responsibility gives you that ability and dependency inversion creates seams throughout your code to spin off in unpredictable directions. These seams are the key to longevity and the difference between enterprise software and regular software.

I’ll share an experience I had this weekend. I was about to deploy a new application and I thought it would be cool to use a performance counter to watch throughput. Within about 20 minutes I was able to add a performance counter by only modifying the `App.config` and this was the result:

{% img plain /images/posts/perfmon3.png %}

This was possible because of IoC among other things. The app is composed of many small pieces and you’d be hard pressed to find the `new` keyword where it matters. `SyncListingJob` was already being built-up by Unity Container, so I was able take advantage of an off-the-shelf performance monitor aspect in the Enterprise Library 4.1 Policy Injection Application Block and wire it in via the `App.config`.