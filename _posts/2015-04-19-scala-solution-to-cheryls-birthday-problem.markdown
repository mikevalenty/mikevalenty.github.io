---
title: "Scala Solution to Cheryl’s Birthday Problem"
date: 2015-04-19 14:54:00 -0700
layout: post
categories:
---

I ran across a [Java 8 solution](https://github.com/mad4j/puzzles/blob/master/src/dolmisani/puzzles/CherylBirthday.java) to [Cheryl’s birthday problem](http://www.nytimes.com/2015/04/15/science/a-math-problem-from-singapore-goes-viral-when-is-cheryls-birthday.html) and just had to write a Scala version since this is one of those problems the language was made for. Also, I’ve been on the Java train for a few months and this was a good excuse to brush off the dust and work on my Scala chops.

#### Cheryl’s Birthday Problem

> Albert and Bernard just become friends with Cheryl, and they want to know when her birthday is. Cheryl gives them a list of 10 possible dates.

    May 15, May 16, May 19
    June 17, June 18
    July 14, July 16
    August 14, August 15, August 17

> Cheryl then tells Albert and Bernard separately the month and the day of her birthday respectively.

**Albert:** I don’t know when Cheryl’s birthday is, but I know that Bernard does not know too.

**Bernard:** At first I don’t know when Cheryl’s birthday is, but I know now.

**Albert:** Then I also know when Cheryl’s birthday is.

So when is Cheryl’s birthday?

```scala
@RunWith(classOf[JUnitRunner])
class BirthdayProblem extends FunSuite with ShouldMatchers {

  case class Birthday(month: Int, day: Int)

  object Birthday {
    def apply(s: String): Birthday = s.split("-").map(_.toInt) match {
      case Array(m, d) => Birthday(m, d)
    }
  }

  val birthdays: List[Birthday] = List(
    "5-15", "5-16", "5-19",
    "6-17", "6-18",
    "7-14", "7-16",
    "8-14", "8-15", "8-17"
  ).map(Birthday.apply)

  def uniqueBy[A, K](fn: (A) => K)(b: List[A]): List[A] = {
    def groupSizeIs(size: Int)(x: (_, List[_])) = x._2.size == size
    b.groupBy(fn).filter(groupSizeIs(1)).flatMap(_._2).toList
  }

  val uniqueByDay: (List[Birthday]) => List[Birthday] = uniqueBy(_.day)

  val uniqueByMonth: (List[Birthday]) => List[Birthday] = uniqueBy(_.month)

  // Albert tells us that Bernard doesn't know the answer, so we
  // know the answer must be in a month that does not contain a
  // birthday with a unique day.
  val clue1: List[Birthday] = {
    val monthsWithUniqueDay: Set[Int] = uniqueByDay(birthdays).map(_.month).toSet
    birthdays.filterNot(b => monthsWithUniqueDay.contains(b.month))
  }

  // Since Bernard now knows the answer, that tells us that the
  // day must be unique among the remaining birthdays.
  val clue2: List[Birthday] = uniqueByDay(clue1)

  // Since Albert now knows the answer, that tells us the answer
  // has to be unique by month.
  val clue3: List[Birthday] = uniqueByMonth(clue2)

  // The only remaining birthday is the answer.
  val answer = clue3.head

  test("Cheryl's birthday") {

    answer should equal(Birthday("xx-xx")) // REDACTED

  }
}
```

Yay.
