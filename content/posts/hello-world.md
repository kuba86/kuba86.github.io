+++
draft = false
authors = ["kuba86"]
date = "2021-05-10"
title = "Hello World!"
description = "This is lovely description of the post"
tags = [
"scala",
]
series = []
categories = []
slug = ""
externalLink = ""
+++
# Hello World
This is a test page

#### Code block with backticks

{{< notice tip >}}
Remember to write code and not simply copy and paste! 😉
{{< /notice >}}

```scala
import scala.util.Random

case class Person(name: String, age: Int) {
  val greet: String = s"Hi $name :-)"
}
val jacob: Person = Person("Jacob", 35)
jacob.greet
```
{{< highlight scala "style=monokai" >}}
import scala.util.Random

case class Person(name: String, age: Int) {
val greet: String = s"Hi $name :-)"
}
val jacob: Person = Person("Jacob", 35)
jacob.greet
{{< /highlight >}}