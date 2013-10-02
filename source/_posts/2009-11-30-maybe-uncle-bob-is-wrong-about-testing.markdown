---
layout: post
title: "Maybe Uncle Bob is Wrong About Testing"
date: 2009-11-30 21:45
comments: true
categories: [NHibernate, TDD]
---

As pressure increases, testing decreases. As the testing decreases, errors increase and as errors increase, pressure increases more. This is what’s known as a positive feedback loop, and this was how I spent the better part of last week.

{% img /images/posts/feedback2_thumb.png %}

**Figure A.5** Not enough time to test reduces the available time _Test-Driven Development: By Example by Kent Beck_

It didn’t start out that way. In fact the first few lines of code I wrote were tests. The problem was I could only get about an hour or two in each night and every day that passed we were leaving money on the table.

Since I didn’t have much time at night, I didn’t want to spend it writing tests. I just wanted to get the app done. All I needed to do was sync business listings from an affiliate with GarageCommerce.com, the project I’ve been working on for the last few months. There were only a handful of business rules to deal with and I thought I could just bang it out.

{% img left /images/posts/district9poster_thumb.jpg %}

I know what you’re thinking. You already know where this is headed and you might as well just skip to the end. It’s like in that movie District 9. It started out pretty good with some edgy social commentary but by the end of the movie it just phoned in a cliché Hollywood plot. You know, like “fish out of water”, “buddy movie”, or “trading places” as was the case with District 9.

Anyway, I finished the app and since I only had a few tests I figured I should trace through things a few times to look for obvious problems. After doing that for a few minutes I was ready to press F5 and start working the kinks out. So after hours of zero feedback coding, I took a deep breath and the started the app for the first time.

It promptly bombed out as expected. I worked through a handful of errors and then it was good to go. As I watched it run, I thought to myself that maybe Uncle Bob is wrong about testing and Spolsky isn’t such a douche bag after all. However after processing a couple hundred jobs, it started throwing crazy errors in a tight loop. Crap!

Fast forward a few hours and I’ve got a hunch as to what is causing the problem. The way I wired up the app, I was creating a single NHibernate session for the entire batch of jobs. I knew this could be trouble so I at least made a point to commit after each job and clear the session so it wouldn’t get bogged down with 10K+ entities. I knew the single session was smelly and the errors I was seeing prompted me to refactor.

{% img plain /images/posts/MASHtvshow11_thumb.jpg %}

This is where it got ugly. I needed to do a pretty major overhaul and I was mostly test-less. My blood pressure went way up with each change and the fact that I didn’t have tests just made me take larger steps with no safety net. I felt like a M.A.S.H. surgeon with guts all over the operating table. Part of me thought I should take a few steps back and regroup, but another part of me was reminded of this quote:

> If you’re going through hell, keep going. – Winston Churchill

So I stuck with it and got ‘er done, but it wasn’t much fun. What did I take from this experience? No revelations really, just some common sense reinforcement. It’s like rediscovering that diet and exercise is the key to losing weight, except that I rediscovered that not having tests really sucks when you have to make changes.