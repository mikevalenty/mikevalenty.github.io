---
layout: post
title: "Unit testing scalding jobs"
date: 2014-03-15 15:27
comments: true
categories: 
published: false
---

Consider this scalding job:

```scala
class ComplicatedJob(args: Args) extends Job(args) {

  val bloatedTsv = Tsv("input",
    ('timestamp,
      'host,
      'referer,
      'remote_addr,
      'user_agent,
      'cookie,
      'connection,
      'accept_encoding,
      'accept_language)).read

  bloatedTsv
    .map('timestamp -> 'timestamp) { ts: String => toDateTime(ts) }
    .filter('timestamp) { ts: DateTime => ts.isAfter(DateTime.now.minusDays(30)) }
    .map('remote_addr -> 'country) { ip: String => toCountry(ip) getOrElse "unknown" }
    .filter('country) { country: String => country == "US" }
    .map('user_agent -> 'browser) { userAgent: String => toBrowser(userAgent) }
    .map('accept_encoding -> 'gzip) { enc: String => enc.contains("gzip") }

    // I want to test this block and not deal with other fields
    .map('gzip -> 'gzip) { gzip: Boolean => if (gzip) "yes" else "no" }
    .groupBy('browser, 'gzip) { _.size('count) }
    .groupBy('browser) { _.pivot(('gzip, 'count) ->('yes, 'no)) }
    // I want to test this block

    .write(Tsv("output"))

  ...
}
```
This looks better

```scala
bloatedTsv
  .timestampIsAfter(DateTime.now.minusDays(30))
  .countryEquals("US")
  .doesBrowserSupportGzip()
  .countGzipByBrowser()
  .write(Tsv("output"))
```

This can be done with extension methods

```scala
import Dsl._

object ComplicatedJob {

  implicit class ComplicatedJobRichPipe(pipe: Pipe) {

    // this block is testable in isolation
    def countGzipByBrowser(): Pipe = {
      pipe
        .map('gzip -> 'gzip) { gzip: Boolean => if (gzip) "yes" else "no" }
        .groupBy('browser, 'gzip) { _.size('count) }
        .groupBy('browser) { _.pivot(('gzip, 'count) ->('yes, 'no)) }
    }

    ...
  }

}
```
And this is the test class for it

```scala
@RunWith(classOf[JUnitRunner])
class ComplicatedJobTests extends FunSuite with ShouldMatchers {

  test("should count and pivot rows into columns") {

    val input = List[InputTuple](
      ("firefox", true),
      ("firefox", false),
      ("chrome", true),
      ("safari", false)
    )

    val expected = List[OutputTuple](
      ("firefox", 1, 1),
      ("chrome", 1, 0),
      ("safari", 0, 1)
    )

    countGzipByBrowser(input) { _ should equal(expected) }
  }

}

object ComplicatedJobTests {
  type InputTuple = (String, Boolean)
  type OutputTuple = (String, Int, Int)

  def countGzipByBrowser(input: List[InputTuple])(fn: List[OutputTuple] => Unit) {
    import Dsl._
    JobTest[CountGzipByBrowser]
      .source(Tsv("input", ('browser, 'gzip)), input)
      .sink[OutputTuple](Tsv("output")) { b => fn(b.toList) }
      .run
      .finish
  }

  class CountGzipByBrowser(args: Args) extends Job(args) {

    Tsv("input", ('browser, 'gzip))
      .read
      .countGzipByBrowser()
      .project('browser, 'yes, 'no)
      .write(Tsv("output"))

  }

}
```