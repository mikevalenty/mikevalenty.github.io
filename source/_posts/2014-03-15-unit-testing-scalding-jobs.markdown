---
layout: post
title: "Unit testing scalding jobs"
date: 2014-03-15 15:27
comments: true
categories: 
---
[Scalding](https://github.com/twitter/scalding) is a powerful framework for writing complex data processing applications on Apache Hadoop. It's concise and expressive - almost to a fault. It's dangerously easy to pack gobs of subtle business logic into just a few lines of code. If you're writing real data processing applications and not just ad-hoc reports, unit testing is a must. However tests can get unwieldy to manage as job complexity grows and the arity of data increases.

For example, consider this scalding job:

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
    .map('country -> 'country) { c: String => if (c == "us") c else "other" }
    .groupBy('browser, 'country) { _.size('count) }
    .groupBy('browser) { _.pivot(('country, 'count) ->('us, 'other)) }
    .write(Tsv("output"))

  def toDateTime(ts: String): DateTime = { ... }

  ...
}
```

Testing this job end-to-end would be fragile because there is so much going on and it would be tedious and noisy to build fake data to isolate and highlight edge cases. The pivot operations on lines 20-22 only deal with `browser` and `country` yet test data with all 10 fields is required including valid timestamps and user agents just to get to the pivot logic.

 There are a few ways to tackle this and an approach I like is to use extension methods to breakdown the logic into smaller chunks of testable code. The result might look something like this.

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

Each block of code depends on only a few fields so it doesn't require mocking the entire input set.

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

In this example only `browser` and `country` are required so setting up test data is reasonably painless and the intent of the test case isn't lost in a sea of tuples. Granted, this approach requires creating a helper job to set up the input and capture the output for test assertions, but I think it's a worthwhile trade off to reveal such a clear test case.

```scala
import ComplicatedJob._
import ComplicatedJobTests._

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