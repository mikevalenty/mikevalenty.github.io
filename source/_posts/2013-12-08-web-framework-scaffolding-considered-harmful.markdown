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

<div style="clear:both;"></div>

### Revision history

{% img right /images/posts/example_rev.png 300 %}

Google docs keeps a revision history of what was changed, when and by whom. That's a pretty cool feature and one you're not likely to build when there are more pressing customer facing features to work on.

Sometimes release notes aren't detailed enough when trying to correlate a surprising change in a KPI, so it's reassuring to have the nitty gritty revision history there when you need to dig a little deeper. It's just like reviewing source control history when tracking down a bug that was introduced in the latest code release. Your data deserves the same respect.

### Concurrent editing

My older brother carries a pocket knife at all times, and when he believes in something, he can't help but sell everyone else on the idea. He used to tell me I needed to carry a pocket knife so I could cut off my seatbelt when trapped in a burning (or sinking) car. In his defense, [GM recalled 240,000 SUVs in 2010](http://money.cnn.com/2010/08/17/autos/gm_recall/index.htm) citing "...[the seat belt] may seem to be jammed."

I eventually succumbed to the peer pressure and started carrying a pocket knife too. I figured that along with untrapping myself from a burning car or defending myself at an ATM, it would be handy when I needed to open a box. To my surprise, new opportunities presented themselves and I was using my knife several times a day. I could cut off a hanging thread, scrape bee pollen off my windsheild, improve the vent in my coffee lid, remove a scratchy tag from my son's shirt and yes, open a box.

It turns out I had a similar experience with concurrent editing. Working on the same document at the same time with multiple people is pretty cool, and you'd be surprised how this powerful ability can change the way you work. There's no way you would justify an extravagant feature like this in your own back office tools and you get it for free with Google docs.

### Formulas

Since it's a spreadsheet, you can use formulas and this is cooler than you might think. Consider a list of products. Chances are the product prices have some kind of relationship that you can capture with a formula and save yourself from tedious and error prone manual entry.

In reality though, you probably don't care about the actual product prices as much as you care about your margins or some other derived number. With a spreadsheet, you can keep your formula together with your data. When you change a price, you can see how it affects your bottom line in a calculated field a few cells over. Or work backwards and calculate the price based on a combination of other factors. Or just use a forumla as a sanity check that all your numbers add up to the right amount.

Again, this is the kind of feature that invites new ways of working. You might use a formula to find a bug before the data ever hits your app or add a new forumla so you don't get burned twice by the same mistake. A spreadsheet gives you a place to capture knowledge about your data. You could capture this knowledge in a wiki, but the fact that it's in a Google spreadsheet that's connected to your app, makes it real. It's the difference between a code comment and a unit test.

### Import

{% img right /images/posts/example_dup.png 300 %}

A data update is a deployment just like a code deployment and things can go horribly wrong when there's a mistake in your data. Big companies get this and respond with change advisory boards and more process to protect themselves, but you can do better.

Your data deserves a repeatable one-click deployment process and you shouldn't settle for anything less. The Google docs API is robust and easy to use. You can fairly easily build a tool that lists all the spreadsheets along with metadata like last modified. Oh, and if you spin off a copy prior to making changes, you get one-click rollback too.

### Conclusion

Empower your team (not just developers) with a familiar tool like a spreadsheet and give them the opportunity to impress you. Watch as calculations and charts emerge to add deeper meaning to your data. Watch as collaboration occurs and team members are brought together rather than divided by functional roles.

Seriously. Don't waste your time half-assing back office tools when you can invest a similar amount of effort in a tool to import data from Google spreadsheets. A little bit of discipline with a scrappy tool and you're paying attention to what matters.

<div style="clear:both;"></div>

> If you can do a half-assed job of anything, you’re a one-eyed man in a kingdom of the blind. – Kurt Vonnegut
