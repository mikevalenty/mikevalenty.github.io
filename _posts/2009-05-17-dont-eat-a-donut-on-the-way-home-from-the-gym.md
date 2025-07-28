---
title: "Don’t Eat a Donut on the Way Home From the Gym"
date: 2009-05-17 21:32:00 -0700
layout: post
categories:
- tdd
---

<img class="right" src="/images/posts/homer-donut-thumb.jpg">

I must warn you, this post is about unit testing software, not donuts. With that said, I’ve been pondering the dilemma of how much time to spend writing unit tests and today I had a moment of clarity that I couldn’t help but share.

I was pairing with a coworker on Friday and we were both feeling a bit unproductive because we had spent so much time writing unit tests for a relatively small bit of code. In fact we spent more time writing tests than we spent writing the code to make the tests pass. To make matters worse, we spent more time refactoring the unit tests than we did refactoring the code that made the tests pass. We were pining over the readability of the tests as if the test were more important than the code and that left me with an uneasy feeling over the weekend.

Today I found a bit of inner peace as I embraced the the notion that the tests are more important than the code that makes them pass. Think about that for a moment. The challenge in writing software is not the implementation. Syntax, structures and algorithms are the easy parts. The real hard earned knowledge won through experience comes in the form of specifications; Understanding the intricate complexities and desired behavior of your software in the wild.

Writing software can be like swimming in the ocean when a thick fog rolls in. The mechanics of swimming are learned easily, but the real challenge is knowing which direction to swim so that you get back to the shore before you run out of energy. After a day of programming, the real asset is not the implementation of your feature, it’ s the increased understanding of your problem domain. This understanding is captured in the form of executable requirements.

When there is a bug in your system, chances are it’s a bug in your understanding of how your system should behave under a particular set of conditions. Burying an innocuous if statement in the middle of some method deep in the stack is a horrible way to reap the reward of day spent spelunking through code. It’s like eating a donut on the way home from the gym.

The legacy you leave is the the unit test, it tells the story of the hard fought knowledge and the readability of your test is more important than the readability of your implementation. Tell the next developer what’s important through an executable specification that reads like one.
