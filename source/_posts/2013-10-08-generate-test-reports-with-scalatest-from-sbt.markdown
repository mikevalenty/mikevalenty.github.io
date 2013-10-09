---
layout: post
title: "Generate Test Reports with ScalaTest from SBT"
date: 2013-10-08 16:42
comments: true
categories: [scala]
published: false
---
The ScalaTest [documentation](http://www.scalatest.org/user_guide/using_the_runner) indicates that xml reports can be generated using the `-u <directory>` commandline argument. Further [documentation](http://www.scalatest.org/user_guide/using_scalatest_with_sbt) claims you can pass arguments from sbt to scalatest, however it is unclear on how to specify an argument that has a value. That's why I'm here.

``` scala
testOptions in Test += Tests.Argument("-u", "target/test-reports")
```

Another thing the documentation omits is the fact that this only works in 2.0., which is only available as a release candidate at the time of this post.

``` scala
libraryDependencies ++= Seq(
    "org.scalatest" %% "scalatest" % "2.0.RC1" % "test",
    ...
)
```