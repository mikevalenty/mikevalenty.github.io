---
layout: post
title: "Definition of Done"
date: 2009-05-30 21:09
comments: true
categories: [Acceptance Testing, Continuous Integration, Technical Debt]
---

> If you don’t know where you are going, you will wind up somewhere else. – Yogi Berra

When asked how a project is going, most programmers will offer one of two discreet responses. It’s either “I just started looking at the code” or “I’m done, I just need to clean up a few things.” Upon further investigation, I have found that done means the hard part has been figured out and there is usually an IDE output window or a funky test web page running on localhost that can demonstrate this status.

The problem is that the hard part is really the fun part, and the actual hart part is the “…I just need to clean up a few things.” So, to remind me and the rest of the team what done means, we have the following definition posted prominently on the wall.

### 1) Unit Tested

This doesn’t need much of an explanation, but having a formal definition posted on the wall is a good reminder for a team new to unit testing.

Unit testing by itself is important, but the real boost comes from using a CI server. We use TeamCity for our .NET projects and phpUnderControl for php projects.

### 2) Acceptance Tested

For our team, acceptance testing means that we deploy our new code to a demo server and write Selenium tests for it. We export the selenium tests as phpUnit fixtures that CruiseControl will run whenever our svn repository is updated. Before the selenium tests are run, we need to update our demo server, so we have CruiseControl call `http://ourdemoserver.com/svn-update.php` first.

### 3) Packaged For Deployment

For our .NET projects, we use a homegrown tool for packaging and deploying. It can stop/start IIS register/unregister COM+ objects and rollback across a farm. For php projects, we simply hand roll a zip file and use some lightweight scripts to unpack the files on the server with rollback ability.

### 4) No Increased Technical Debt

I ask myself if this code is going to be an asset that makes us stronger and able to respond more quickly to future business opportunities, or is it going to be a fragile liability that I will need to carefully tip-toe around 30 seconds after it’s deployed.

Just like a parent thinks their kid is the cutest kid ever, it can be hard to look at your own work when you’ve got your head wrapped around it and come to terms with the fact that you’re about to deploy some legacy code. I usually grab a coworker and walk through things while paying close attention to the [only valid measurement of code quality](http://www.osnews.com/story/19266/WTFs_m).

{% img /images/posts/wtfm.jpg %}