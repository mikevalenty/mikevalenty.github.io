---
title: "5 Ways to Fail at Pair Programming"
date: 2009-05-10 21:37:00 -0700
layout: post
categories:
- pair programming
- tdd
---

<img class="plain" src="/images/posts/shoes.jpg">

The only hard part is convincing your boss that it’s not a colossal waste of time, right? Well that’s the first hurdle and yes it’s a doosie, but it’s just the price of admission. The real fun starts when you sit next to your partner and try to prove your boss wrong. After a year or so in the trenches, I’ve learned a thing or two on how to mess it up.

### 5. Researching new technology

“Click on that link, no wait scroll up, over there, wait I wasn’t done reading that…” That’s not pair programming, that’s insanity. Recently we were working on a project that involved a custom Firefox browser skin. Neither of us had experience with browser skins, so there was a lot of Googling and tutorial reading. As soon as we caught ourselves reading blogs and whatnot, we would split up for 15 minutes and then compare notes.

### 4. Getting your environment going

“Hey buddy, are you ready to start pairing on that project? Let’s see, now where is the source code for the project we’re working on… Hmm, this doesn’t compile – I think I need to install the latest version of that library.” Kill me now.

### 3. Paralysis

<img class="plain right" src="/images/posts/FrightenedMan400.jpg">

One of my favorite authors, Kurt Vonnegut would write clever things like “The man looked… guilty isn’t the right word, but it’s the first one that comes to mind.”

With pair programming, you can’t just sit there because you don’t know where to start or you can’t think of the right variable name. Just type something. You have to get the problem solving out of your head and into a medium that two people can have a conversation about it. Maybe people are afraid to type something in front of a peer if it’s not perfect. Borrow a technique from Kurt Vonnegut and as you are type, just say “I don’t want the code to look like this…” and write some awful switch case statement.

Once there is something on the screen, you and your partner are instantly aligned and solving the same problem. Now you can discuss factories and polymorphism and all kinds of heady solutions.

### 2. Not doing TDD

Have you ever watched an artist paint a picture and thought to yourself “what the heck is that”. Then with the next brush stroke you realize you’re looking at the profile of someone’s face. Well watching someone write code can be horribly worse than that.

Pair programming is a conversation, not a seminar or window into someone’s brain. The best way to keep things at a conversational level is to work top down which is exactly what TDD forces you to do. You must consider your code from the client perspective and challenge yourself to keep your code on purpose.

Even more importantly, TDD gives you and your buddy an “out” at regular intervals. You write a test, you make a test pass. Either way, you have many small victories over the course of a few hours and you can switch pairs at well defined feel-good stopping points.

### 1. Not using a timer

<img class="plain left" src="/images/posts/pprobot.jpg">

I used to work with a guy that was always competing with something. A mutual friend at the office was doing the 3-day breast cancer walk. If anyone in the office donated money, he would up his donation $1 higher. On the softball team, his jersey number was #1. When we started playing ping-pong at lunch, he bought a $600 ping-pong robot to practice with at home. So when he started using a timer while programming I just figured it was good old Paul racing against time.

Then I paired with him on a project. Right after wanting to strangle him for not having his environment ready, we set his timer for 30 minutes and got going. When the timer went off, we stopped mid keystroke and switched seats. This did a few things. First, it kept Paul engaged the whole time because he knew he would be in the hot seat in a few minutes, and I can be a bit of a bully so it was easier for me to back off knowing I’d get my chance soon enough. One of the more surprising benefits, however, was simply the fact that we would sit for a minute and agree exactly what we were doing before starting the timer which really got us in gear and drastically cut down on tangents.

Using a timer creates a magical dynamic that I cannot do justice with words. It’s like TDD, it’s not about the tests – it’s about the code you write to make it testable. You have to try it. Period.

> If you can do a half-assed job of anything, you’re a one-eyed man in a kingdom of the blind. – Kurt Vonnegut
