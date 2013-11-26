---
layout: post
title: "Using Scala to Perform a Multi-Key Get on a Couchbase View"
date: 2013-11-16 15:26
comments: true
categories: 
---
To retrieve documents from Couchbase by anything other than the document key requires querying a [view](http://www.couchbase.com/docs//couchbase-manual-2.0/couchbase-views.html) and views are defined by map and reduce functions written in JavaScript. Consider a view that returns data like this:

``` javascript
key         value
---         -----
"key1"      { "term": "red", "count": 2 }
"key2"      { "term": "red", "count": 1 }
"key3"      { "term": "blue", "count": 4 }
...
```

And this Scala `case class` to hold the documents retrieved from the view.

``` scala
case class TermOccurrence(term: String, count: Int)
```

It's a common scenario to retrieve multiple documents at once and the [Java driver](http://www.couchbase.com/communities/java/getting-started) has a pretty straight forward api for that. The desired keys are simply specified as a json array.

``` scala
import com.couchbase.client.CouchbaseClient
import play.api.libs.json.Json
import CouchbaseExtensions._

@Log
def findByTerms(terms: List[String]): List[TermOccurrence] = {

  val keys = Json.stringify(Json.toJson(terms map (_.toLowerCase)))

  val view = client.getView("view1", "by_term")
  val query = new Query()
  query.setIncludeDocs(false)
  query.setKeys(keys)

  val response = client.query(view, query).asScala

  response.toList map (_.as[TermOccurrence])
}
```

The Java driver deals with strings so it's up to the client application to handle the json parsing. That was an excellent design decision and makes using the Java driver from Scala pretty painless. I'm using the [Play Framework](http://www.playframework.com/) json libraries and an extension method `_.as[TermOccurrence]` defined on `ViewRow` to simplify the mapping of the response documents to Scala objects.

``` scala
object CouchbaseExtensions {

  class ViewRowExtensions(row: ViewRow) {
    def as[A](implicit format: Format[A]): A = {
      val document = row.getValue
      val modelJsValue = Json.parse(document)
      Json.fromJson[A](modelJsValue).get
    }
  }

  implicit def enhanceViewRow(row: ViewRow) = new ViewRowExtensions(row)
}
```
In order for this extension method to work, it requires an implicit `Format[TermOccurrence]` which is defined on of the `TermOccurrence` compainion object.

``` scala
object TermOccurrence {
  implicit val formatTermOccurrence = Json.format[TermOccurrence]
}
```