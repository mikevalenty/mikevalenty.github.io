---
layout: post
title: "Reading Google Spreadsheets with Scala"
date: 2013-12-08 09:59
comments: true
categories: 
published: false
---

If you're a start up and spending time building back office tools to input application data into web forms, you're doing it wrong. Sure, it's a pretty simple task to build a few [CRUD](http://en.wikipedia.org/wiki/Create,_read,_update_and_delete) screens using the hottest new MVC framework, but you can do better.

Integrating with Google spreadsheets to manage application data unlocks many possibilities and will take you further than anything you could scaffold up in a comperable amount of time. To be clear, I'm not suggesting you take a runtime dependency on Google docs. I'm talking about using a light weight business process that takes advantage of the rich collaborations features of Google docs and simply *import* the data into your database using the [API](https://developers.google.com/google-apps/spreadsheets/).

{% img /images/posts/example_sp.png 600 %}

### Access control

I'm sure your web framework has some sort of access control baked in, but Google's is probably better. You can invite collaborators and control their level of access to each document. When you grant someone access, they get notified via email.

### Revision history

{% img right /images/posts/example_rev.png 300 %}

When collaborators make changes, Google keeps a revision history of what was changed, when and by whom. That's a pretty cool feature and one you're not likely to build when there are more pressing customer facing features to work on.

### Concurrent editing

You can work on the same document at the same time with multiple people and not clobber each other's changes. You don't need this, but you get it for free.

### Bulk operations

Since it's a spreadsheet, you can use formulas. This is cooler than you might think. Consider a list of products. Chances are the product prices have some kind of relationship that you can caputre with a formula and save yourself from tedious and error prone individual entries. Sure a developer can write some SQL or whatever query language you use, but you could empower a team member more directly responsible for managing the product data.

### Repeatable

This is easily the most important feature. Once your custom import tool is working, you can repeatedly import data into a test environment, make changes and repeat the process until you're confident enough to run the same process in production. This is a big deal and you should pause to let it sink in for a minute.

A data update is a deployment just like a code deployment. Big companies get this and assemble change advisory boards to make sure everybody did their due dilligence. 

--

{% img right /images/posts/example_dup.png 300 %}

Oh, and it's a good idea to spin off a snapshot of the spreadsheet prior to making changes so you can rollback easily without having to wade through the detailed revision history.

Seriously. Don't waste your time half-assing back office tools when you can invest a similar amount of programming effort in a simple tool to import data from Google spreadsheets into your database.

<div style="clear:both;"></div>

> If you can do a half-assed job of anything, you’re a one-eyed man in a kingdom of the blind. – Kurt Vonnegut
