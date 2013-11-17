---
layout: post
title: "User Recommendations with Couchbase and Scala"
date: 2013-11-16 13:29
comments: true
categories: 
published: false
---

Here's how I did some recommendations in scala:

This is the raw data stored in couchbase

``` javascript
{
   "id": "...",
   "reports": [
       {
           "title": "Games",
           "terms": [
               "call of duty",
               "battlefield",
               "grand theft auto",
               "mine craft",
               "madden"
           ]
       }
   ]
}
```

It looks like:

``` javascript
key             value
---             -----
"call of duty"  { "term": "grand theft auto", "count": 1 }
"call of duty"  { "term": "battlefield", "count": 1 }
"call of duty"  { "term": "mine craft", "count": 1 }
"call of duty"  { "term": "madden", "count": 1 }
"battlefield"   { "term": "call of duty", "count": 1 }
"battlefield"   { "term": "grand theft auto", "count": 1 }
"battlefield"   { "term": "mine craft", "count": 1 }
...
```

Now it has to be rolled up

``` scala
case class TermOccurrence(term: String, count: Int)
```

``` scala
@Log
def recommendations(terms: List[String]): List[TermOccurrence] = {

  val occurrences: List[TermOccurrence] = repository.findByTerms(terms)

  val recs = occurrences groupBy (_.term) map { o =>
    val total = (o._2 map (_.count)).sum
    TermOccurrence(o._1, total)
  }

  def substringOfTerm(o: TermOccurrence) =
    terms exists (t => t.contains(o.term) || o.term.contains(t))

  def substringOfRec(o: TermOccurrence) =
    recs exists (r => r != o && r.term.contains(o.term))

  recs.toList
    .filterNot(o => substringOfTerm(o) || substringOfRec(o))
    .sortBy(-_.count)
    .take(5)
}
```