---
title: "Scala Solution to Finding Happy Numbers"
date: 2015-05-03 10:45:00 -0700
layout: post
categories:
---

What is a Happy Number?

> Starting with any positive integer, replace the number by the sum of the squares of its digits, and repeat the process until the number equals 1 (where it will stay), or it loops endlessly in a cycle which does not include 1.
>
> <http://en.wikipedia.org/wiki/Happy_number>

```scala
@RunWith(classOf[JUnitRunner])
class HappyTests extends FunSuite with ShouldMatchers {

  def happy(x: Int): Boolean = {

    def sumOfSquares(x: Int): Int = {
      def digits(x: Int): Seq[Int] = if (x < 10) Seq(x) else digits(x / 10) :+ x % 10
      def squared(x: Int): Int = x * x
      digits(x).map(squared).sum
    }

    def loop(seen: Set[Int], x: Int): Boolean = {
      x match {
        case 1 => true
        case _ if seen.contains(x) => false
        case _ => loop(seen + x, sumOfSquares(x))
      }
    }

    loop(Set(), x)
  }

  test("happy numbers") {

    val actual = 1 to 50 filter happy

    val expected = Seq(1, 7, 10, 13, 19, 23, 28, 31, 32, 44, 49)

    actual should equal(expected)
  }
}
```
