---
title: "Extracting Root Domain From a Url"
date: 2013-10-06 17:05:00 -0700
layout: post
categories:
- scala
---

Given a url like `http://www.google.com/?q=tld+uk`, extract the *root domain*. In this case, it would be `google.com`. Sounds easy, right? Well it is, but `http://www.google.com.br/` is also legit. Okay, so recursion to the rescue!

Not so fast…

    .ac.uk
    .co.uk
    .gov.uk
    .parliament.uk
    .police.uk
    ...
    .metro.tokyo.jp
    .city.(cityname).(prefecturename).jp
    ...

> <http://en.wikipedia.org/wiki/.uk>
> <http://en.wikipedia.org/wiki/.jp>

Doh! Surely [George Clinton](http://en.wikipedia.org/wiki/Parliament_(band)) is not happy about this and neither am I because it means I’m stuck doing a lookup against a list of arbitrary domains.

Using `java.net.URI` takes care of extracting the host, so the interesting part is parsing the host into a list of chunks that decrease in number of segments.

```scala
test("should split host into chunks of decreasing parts") {

  val chunks = splitIntoChunks("www.google.com.br")

  chunks.toList should equal(List("www.google.com.br", "google.com.br", "com.br", "br"))
}
```

An implementation of `splitIntoChunks` isn’t terribly exciting, but probably a good interview question. How about an implementation that doesn’t mutate state? Sounds fun, but why make this more difficult? It’s not because I want to run *this* bit of code on mutliple cores or distributed across machines, but because it challenges me to change the way I think about simple problems so that solving more complicated problems using functional idioms feels more natural. After all, when something is painful, you should do it more often. You know, like push ups and deployments.

```scala
def splitIntoChunks(host: String): List[String] = {

  val parts = host.split('.').toList

  parts.dropRight(1).scanRight(parts.last) {(acc, p) => s"$acc.$p"}
}
```

Scala ftw.
