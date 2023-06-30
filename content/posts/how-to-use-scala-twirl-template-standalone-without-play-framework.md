+++
draft = false
authors = ["kuba86"]
date = "2021-06-29"
lastmod = "2023-05-03"
images = ["/posts/how-to-use-scala-twirl-template-standalone-without-play-framework.png"]
title = "How to use Scala Twirl template standalone, without Play Framework"
description = "Twirl is a type safe template engine based on Scala, and designed for Play Framework. It can also be used standalone, without Play."
tags = [
"scala",
"twirl",
"play framework",
]
aliases = [
    "how-to-use-scala-twirl-template-engine-standalone-without-play-framework/"
]
+++
![How to use Scala Twirl template standalone, without Play Framework](/posts/how-to-use-scala-twirl-template-standalone-without-play-framework.png)

## Twirl in Scala
Twirl is a type safe template engine based on Scala, and designed for Play Framework. It can also be used standalone, without Play. I will go through some information about Twirl, how to set it up, and use in a standalone Scala application.

Keep in mind:
1. Templates are compiled to a standard Scala functions. This provides type safety.
1. When used standalone in Scala, Twirl needs to be added as an SBT plugin (sbt-twirl), not as library dependency.
1. If you are using Intellij Idea please note that you will see `Cannot resolve symbol` until SBT compiles your templates. You can always run `sbt compile`

## Setting up SBT project with Twirl and Scala
### Add SBT plugin sbt-twirl
In our project, [add the sbt plugin](https://www.scala-sbt.org/1.x/docs/Using-Plugins.html#Declaring+a+plugin) to `/project/plugins.sbt`:
```scala
addSbtPlugin("com.typesafe.sbt" % "sbt-twirl" % "1.5.1")
```
Then in `/build.sbt`:
```scala
name := "twirl-in-scala-standalone-example"
version := "0.1"
scalaVersion := "2.13.6"

lazy val root = (project in file(".")).enablePlugins(SbtTwirl)
```
If you have more complex setup take a look at [official documentation about enabling and disabling plugins](https://www.scala-sbt.org/1.x/docs/Using-Plugins.html#Enabling+and+disabling+auto+plugins) in SBT.
## Creating new Twirl template
I recently had to generate multiple configuration files for Apache httpd server. My first choice was Apache Module mod_macro, however, it has too many limitations. My second choice was to use [mustache templates](https://github.com/mustache/mustache) and I was super excited that it is also [available in bash!](https://github.com/tests-always-included/mo) Everything seems great until I attempted to use arrays, and it did not work. Most likely because it was implemented for Bash 3.x which is 10 years old. I did not want to mess with my environment by adding or switching to older version of Bash. Because Scala is my weapon of choice, I decided for my third attempt, to try solving this problem using Scala and Twirl templating engine. I know it might seem like an overkill, however, my previous choices have failed me, so I wanted my next attempt to be the final one! 😎

Official documentation, and most tutorials online show how to use it in context of REST API, html files, etc. For this demonstration I will be showing how to use Twirl in Scala for generating Apache httpd configuration files. Whatever your use case I am sure Scala and Twirl will get the job done.

Twirl templates should be placed in `/src/main/twirl` directory, and the file name should be `[name].scala.[extension]`. The extension corresponds to Twirl formats, and out of the box we can choose from HTML, Text, XML, and JavaScript. Because we want to create a template for Apache httpd configuration our file name will be `vHostHttpToHttps.scala.txt`. The purpose for it will be to create `VirtualHost` configuration to redirect insecure http requests to more secure https. The template code will look something like this:
```scala
@(subDomain: String, mainDomain: String, redirectStatusCode: Int)
@fullDomain=@{
  if(subDomain == "") {
    mainDomain
  } else {
    s"$subDomain.$mainDomain"
  }
}
<VirtualHost *:80>
  ServerName @fullDomain
  RewriteEngine On
  RewriteRule (.*) "https://%{HTTP_HOST}%{REQUEST_URI}" [R=@redirectStatusCode,L]
</VirtualHost>
```
Let's dissect the code into smaller pieces. First line defines function parameters:
```scala
@(subDomain: String, mainDomain: String, redirectStatusCode: Int)
```
Lines two to eight, contains a block of Scala code, with result assigned to `fullDomain`
```scala
@fullDomain=@{
  if(subDomain == "") {
    mainDomain
  } else {
    s"$subDomain.$mainDomain"
  }
}
```
Lines nine to thirteen, defines our template of Apache httpd configuration. You can probably spot the @ character, the 'Twirl', from which Twirl template engine got it's name. That's the magic character which indicates dynamic content.
```scala
<VirtualHost *:80>
  ServerName @fullDomain
  RewriteEngine On
  RewriteRule (.*) "https://%{HTTP_HOST}%{REQUEST_URI}" [R=@{redirectStatusCode},L]
</VirtualHost>
```
Two cases where we have dynamic content is for ServerName and for status code for redirection. You can write them with just the magic character or surround them with curly brackets. In mustache templates that would be `{{ fullDomain }}` where in Twirl we have `@fullDomain` or `@{fullDomain}`.
## Using Twirl template in Scala code
Now we are ready to use the template in Scala code. For that, lets create a new Scala file `Generator.scala` in `/src/main/scala` directory. The simplest case would be to print out the result to console. In that case our Generator will look something like this:
```scala
object Generator extends App {
  println(
    txt.vHostHttpToHttps(
      subDomain = "blog",
      mainDomain = "kuba86.com",
      redirectStatusCode = 301
    )
  )
}
```
Lines three to six are most interesting to us. We start with extension `txt`, then our function name `vHostHttpToHttps`, and last, we provide function parameters which are defined on the first line of our Twirl template. When you run the above code, SBT will compile Twirl templates, and the Generator object will hopefully print out the result:
```text
<VirtualHost *:80>
  ServerName blog.kuba86.com
  RewriteEngine On
  RewriteRule (.*) "https://%{HTTP_HOST}%{REQUEST_URI}" [R=301,L]
</VirtualHost>
```
You can check out `/target/scala-2.13/twirl/main/txt` directory and see Scala code that was generated by sbt-twirl plugin. The file name will be `[name].template.scala` in our case `vHostHttpToHttps.template.scala` will look like this:
```scala
package txt

import _root_.play.twirl.api.TwirlFeatureImports._
import _root_.play.twirl.api.TwirlHelperImports._
import _root_.play.twirl.api.Html
import _root_.play.twirl.api.JavaScript
import _root_.play.twirl.api.Txt
import _root_.play.twirl.api.Xml

object vHostHttpToHttps extends _root_.play.twirl.api.BaseScalaTemplate[play.twirl.api.TxtFormat.Appendable,_root_.play.twirl.api.Format[play.twirl.api.TxtFormat.Appendable]](play.twirl.api.TxtFormat) with _root_.play.twirl.api.Template3[String,String,Int,play.twirl.api.TxtFormat.Appendable] {

  /**/
  def apply/*1.2*/(subDomain: String, mainDomain: String, redirectStatusCode: Int):play.twirl.api.TxtFormat.Appendable = {
    _display_ {
      {

def /*2.2*/fullDomain/*2.12*/ = {{
  if(subDomain == "") {
    mainDomain
  } else {
    s"$subDomain.$mainDomain"
  }
}};
Seq[Any](format.raw/*1.66*/("""
"""),format.raw/*8.2*/("""
"""),format.raw/*9.1*/("""<VirtualHost *:80>
  ServerName """),_display_(/*10.15*/fullDomain),format.raw/*10.25*/("""
  """),format.raw/*11.3*/("""RewriteEngine On
  RewriteRule (.*) "https://%"""),format.raw/*12.30*/("""{"""),format.raw/*12.31*/("""HTTP_HOST"""),format.raw/*12.40*/("""}"""),format.raw/*12.41*/("""%"""),format.raw/*12.42*/("""{"""),format.raw/*12.43*/("""REQUEST_URI"""),format.raw/*12.54*/("""}"""),format.raw/*12.55*/("""" [R="""),_display_(/*12.61*/redirectStatusCode),format.raw/*12.79*/(""",L]
</VirtualHost>
"""))
      }
    }
  }

  def render(subDomain:String,mainDomain:String,redirectStatusCode:Int): play.twirl.api.TxtFormat.Appendable = apply(subDomain,mainDomain,redirectStatusCode)

  def f:((String,String,Int) => play.twirl.api.TxtFormat.Appendable) = (subDomain,mainDomain,redirectStatusCode) => apply(subDomain,mainDomain,redirectStatusCode)

  def ref: this.type = this

}
```
Because this file is autogenerated by SBT, it should be treated as read only. Don't try to modify it. 
## Summary
Scala Twirl template engine can be use in standalone applications, without the need for Play Framework. I personally like to use it and for my use case I was able to solve the problem faster than trying to figure out Apache httpd mod_macro or mustache. If you like to check out complete project, [you can find it on GitHub](https://github.com/kuba86/twirl-in-scala-standalone-example). Enjoy!
