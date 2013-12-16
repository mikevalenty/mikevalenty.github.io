---
layout: post
title: "Web Framework Scaffolding Considered Harmful"
date: 2013-12-08 09:59
comments: true
categories: 
published: false
---

If you're a start up and spending time building back office tools to input application data into web forms, you're doing it wrong. Sure, it's a pretty simple task to build a few [CRUD](http://en.wikipedia.org/wiki/Create,_read,_update_and_delete) screens using the hottest new MVC framework, but you can do better.

Integrating with Google spreadsheets to manage application data unlocks powerful possibilities and will take you further than anything you could scaffold up in a comperable amount of time. To be clear, I'm not suggesting you take a runtime dependency on Google docs. I'm talking about using a lightweight business process that takes advantage of the rich collaborations features of Google docs and simply *import* the data into your database using the [API](https://developers.google.com/google-apps/spreadsheets/).

{% img /images/posts/example_sp.png 600 %}

### Access control

I'm sure your web framework has some sort of access control baked in, but Google's is probably better. You can invite collaborators and control their level of access to each document. When you grant someone access, they get notified via email. If you create a gmail account exclusively for your app, anyone on your team can create a spreadsheet and share it with the app account. Your import tool can use the API to list all the spreadsheets available along with metadata like the last modified date.

This may not sound like a big deal, but a simple import tool enables _anyone_ on your team to spin off a copy, make some tweaks and try it out in a test environment. If your process and tools don't empower your entire team, you're squandering your start up's greatest advantage.

<div style="clear:both;"></div>

### Revision history

{% img right /images/posts/example_rev.png 300 %}

Google docs keeps a revision history of what was changed, when and by whom. That's a pretty cool feature and one you're not likely to build when there are more pressing customer facing features to work on.

Sometimes release notes aren't detailed enough when trying to correlate a surprising change in a KPI, so it's reassuring to have the nitty gritty revision history there when you need to dig a little deeper. It's just like reviewing source control history when tracking down a bug that was introduced in the latest code release. Your data deserves the same respect.

### Concurrent editing

My older brother carries a pocket knife at all times, and when he believes in something, he can't help but sell everyone else on the idea. He used to tell me I needed to carry a pocket knife so I could cut off my seatbelt when trapped in a burning (or sinking) car. In his defense, [GM recalled 240,000 SUVs in 2010](http://money.cnn.com/2010/08/17/autos/gm_recall/index.htm) citing "...[the seat belt] may seem to be jammed."

I eventually succumbed to the peer pressure and started carrying a pocket knife too. I figured that along with untrapping myself from a burning car or defending myself at an ATM, it would be handy when I needed to open a box. To my surprise, new opportunities presented themselves and I was using my knife several times a day. I could cut off a hanging thread, scrape bee pollen off my windsheild, improve the vent in my coffee lid, remove a scratchy tag from my son's shirt and yes, open a box.

It turns out I had a similar experience with concurrent editing. Working on the same document at the same time with multiple people is pretty cool, and you'd be surprised how this powerful ability can change the way you work. There's no way you would justify an extravagant feature like this in your own back office tools.

### Bulk operations

Since it's a spreadsheet, you can use formulas. Perhaps this is obvious, but it's cooler than you might think. Consider a list of products. Chances are the product prices have some kind of relationship that you can capture with a formula and save yourself from tedious and error prone individual entries. Sure a developer can write some SQL or whatever query language you use, but you could empower a team member more directly responsible for managing the product data.

### Repeatable

{% img right /images/posts/example_dup.png 300 %}

This is easily the most important feature. A data update is a deployment just like a code deployment. Things can go horribly wrong when there's a bug in your data. Big companies _get this_ and respond with change advisory boards and more process to protect themselves. 

With a small investment in an import tool, you get a repeatable one-click deployment process for your data. Oh, and if you spin off a copy prior to making changes, you get one-click rollback too. It's not big company bureacracy, it's a little bit of disciple with a scrappy tool and you're ahead of the crowd.

<div style="clear:both;"></div>

--

Seriously. Don't waste your time half-assing back office tools when you can invest a similar amount of programming effort in a simple tool to import data from Google spreadsheets.

<div style="clear:both;"></div>

> If you can do a half-assed job of anything, you’re a one-eyed man in a kingdom of the blind. – Kurt Vonnegut
