---
title: "Parsing Json With Defaults in Scala"
date: 2014-01-24 22:57:00 -0800
layout: post
categories:
---

The [json library in Play](http://www.playframework.com/documentation/2.2.x/ScalaJson) does this thing where it explodes if the json you’re reading is missing a property. [Gson](https://code.google.com/p/google-gson/) would happily leave the property null, but that’s just not the scala way. In scala, if something is optional, it’s wrapped in an `Option`. The problem is I’d like to add a property to my data model and I don’t want it to be optional because it’s not.

In Java land, I probably would have added the property and thought about how to deal with migrating old data later, but not in scala. This battle has to be fought right now. That’s what *type safe* means and it’s why the `NullPointerException` has all but died in pure scala apps, not unlike the [carpal tunnel epidemic](http://freakonomics.com/2013/09/12/whatever-happened-to-the-carpal-tunnel-epidemic/).

In my case I’ve got some data in Couchbase and when I add a new property to my data model, I won’t be able to read the old data. What I need to do is transform the json before hydrating the object. Fortunately, this is a snap by using the rich transformation features described in [JSON Coast-to-Coast](http://mandubian.com/2013/01/13/JSON-Coast-to-Coast/).

My approach was to create a subclass of `Reads[A]` called `ReadsWithDefaults[A]` that reads json and uses a transformation to merge default values. It looks like this:

```scala
case class MyObject(color: String, shape: String)

object MyObject {

    val defaults = MyObject("blue", "square")

    implicit val readsMyObject = new ReadsWithDefaults(defaults)(Json.format[MyObject])

    implicit val writesMyObject = new Json.writes[MyObject]
}
```

```scala
import play.api.libs.json._
import play.api.libs.json.Reads._

class ReadsWithDefaults[A](defaults: A)(implicit format: Format[A]) extends Reads[A] {

  private val mergeWithDefaults = {
    val defaultJson = Json.fromJson[JsObject](Json.toJson(defaults)).get
    __.json.update(of[JsObject] map {o => defaultJson ++ o})
  }

  def reads(json: JsValue) = {
    val jsObject = json.transform(mergeWithDefaults).get
    Json.fromJson[A](jsObject)(format)
  }
}
```
