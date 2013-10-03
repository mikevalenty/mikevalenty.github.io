---
layout: post
title: "Martinizing is Not Refactoring"
date: 2009-05-24 21:30
comments: true
categories: [Open Closed Principle, Refactoring]
---

A friend of mine, Keith, used the term martinizing (as in [Uncle Bob](http://www.objectmentor.com/omTeam/martin_r.html)) for the process of [cleaning code](http://www.amazon.com/Clean-Code-Handbook-Software-Craftsmanship/dp/0132350882). The term has taken on a very specific meaning and it’s worth a few words.

Martinizing is similar to refactoring in that it does not change the observable behavior of the code, but the goal is different. When I refactor, I am changing the design, usually in an effort to add a new feature in an [open closed](http://c2.com/cgi/wiki?OpenClosedPrinciple) manner.

When I martinize, I am telling a story. The most important story being the desired behavior, but also a story of the hard earned knowledge acquired along the way. If I spend hours distilling some business concept. I want to leave a trail for the next guy to understand that a simple property assignment or conditional statement isn’t so simple. And of course the perfect way to punctuate that message is with a well written unit test.