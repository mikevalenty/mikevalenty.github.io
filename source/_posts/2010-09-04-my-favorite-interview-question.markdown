---
layout: post
title: "My Favorite Interview Question"
date: 2010-09-04 08:12
comments: true
categories: 
---

Lately I’ve been wrapping up interviews with the question “What makes software maintainable?” I usually catch a glimpse that says “hmm… I’ve never really thought about maintainability” followed by a response to the question “What makes software good?”, which I didn’t actually ask, but most programmers have thought enough about to give an on the spot answer. The bland responses inevitably involve documentation (fail) and maybe something about consistency (meh).

{% img /images/posts/trump_thumb.jpg %}

The underlying question I’m asking is a question you should ask yourself every time you write a line of line of code. That question is “What are you optimizing?” Most programmers have mentally hard-coded the term optimize to mean efficient use of memory and cpu, however it’s a horribly myopic definition. In fact, it’s just plain wrong unless you’re writing an iPhone game or maybe you work at Google and saving a few cpu cycles translates to kilowatts of power and possibly saving a small rain forest.

Let’s see what Wikipedia has to say.

> In computer science, program optimization or software optimization is the process of modifying a software system to make some aspect of it work more efficiently or use fewer resources.[1] In general, a computer program may be optimized so that it executes more rapidly, or is capable of operating with less memory storage or other resources, or draw less power. — http://en.wikipedia.org/wiki/Optimization_(computer_science) 

{% img /images/posts/conserve_thumb1.jpg %}

It’s not wrong, but it reinforces the notion that less is more and the precious resources are physical resources like memory and cpu. Is it better to have fewer files in your project, fewer dlls in your bin directory, or use fewer semicolons? I recently debugged a chain of jQuery statements that was 430 characters long (un-minified) and used one semicolon. Is that optimized?

Consider this definition:

> In mathematics, computer science and economics, optimization, or mathematical programming, refers to choosing the best element from some set of available alternatives. — http://en.wikipedia.org/wiki/Optimization_(mathematics)

This does a better job of capturing the trade-offs we make on a daily basis. I like the idea of choosing the best element rather than consuming the fewest resources. Instead of asking what you’re optimizing, a better question might be what are you maximizing? For a new iPhone game, maybe time to market is your number one concern. I work on enterprise SaaS applications that have a long shelf life so maintainability is a pretty big deal. I strive to maximize readability and testability when I write software. What are you maximizing?