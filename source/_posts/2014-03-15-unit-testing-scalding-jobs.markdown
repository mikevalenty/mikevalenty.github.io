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
    ('user_id,
      'timestamp,
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
    .map('user_agent -> 'browser) { userAgent: String => toBrowser(userAgent) }
    .map('remote_addr -> 'country) { ip: String => toCountry(ip) }

    // I want to test this block
    .map('country -> 'country) { c: String => if (c == "us") c else "other" }
    .groupBy('browser, 'country) { _.size('count) }
    .groupBy('browser) { _.pivot(('country, 'count) ->('us, 'other)) }
    // I want to test this block

    .write(Tsv("output"))
}
```

This looks better

```scala
class ComplicatedJob(args: Args) extends Job(args) {

  ...

  bloatedTsv
    .timestampIsAfter(DateTime.now.minusDays(30))
    .userAgentToBrowser()
    .remoteAddrToCountry()
    .countCountryByBrowser()
    .write(Tsv("output"))
}
```

This can be done with extension methods

```scala
import Dsl._

object ComplicatedJob {

  implicit class ComplicatedJobRichPipe(pipe: Pipe) {

    // this chunk of code is testable in isolation
    def countCountryByBrowser(): Pipe = {
      pipe
        .map('country -> 'country) { c: String => if (c == "us") c else "other" }
        .groupBy('browser, 'country) { _.size('count) }
        .groupBy('browser) { _.pivot(('country, 'count) ->('us, 'other)) }
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
      ("firefox", "us"),
      ("chrome", "us"),
      ("safari", "us"),
      ("firefox", "us"),
      ("firefox", "br"),
      ("chrome", "de")
    )

    val expected = Set[OutputTuple](
      ("firefox", 2, 1),
      ("safari", 1, 0),
      ("chrome", 1, 1)
    )

    count(input) { _.toSet should equal(expected) }
  }

}

object ComplicatedJobTests {
  type InputTuple = (String, String)
  type OutputTuple = (String, Int, Int)

  // this is a helper job to set up the inputs and outputs
  // for the chunk of code we're trying to test
  class CountCountryByBrowser(args: Args) extends Job(args) {

    Tsv("input", ('browser, 'country))
      .read
      .countCountryByBrowser() // this is what we're testing
      .project('browser, 'us, 'other)
      .write(Tsv("output"))

  }

  // helper method to run our test job
  def count(input: List[InputTuple])(fn: List[OutputTuple] => Unit) {
    import Dsl._
    JobTest[CountCountryByBrowser]
      .source(Tsv("input", ('browser, 'country)), input)
      .sink[OutputTuple](Tsv("output")) { b => fn(b.toList) }
      .run
      .finish
  }
}
```