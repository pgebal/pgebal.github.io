---
layout: post
title:  "Type class pattern"
date:   2016-11-27 11:00:00 +0100
categories: articles
tags: [scala, functional, programming, spray, json, typeclass, type class]
excerpt: An explanation of type class pattern.
comments: true
share: true
---
To get the most out of this article I encourage you to put the pieces of code you find here into a scala worksheet and play with it.


Let's have a class that we want to serialize to JSON:
{% highlight scala %}
case class Color(name: String, red: Int, green: Int, blue: Int)
{% endhighlight %}

If we went Java style, we might come up with this:
{% highlight scala %}
type Json = String

trait SerializableToJson {
  def toJson: Json
}

case class Color(name: String, red: Int, green: Int, blue: Int)
 extends SerializableToJson {
  override def toJson: Json =
    s"{'name': '$name', 'red': $red, 'green': $green, 'blue': $blue}"
}
{% endhighlight %}

Pros:
<ul>
<li> it works. </li>
</ul>
Cons:
<ul>
<li> <b>Color</b> knows how to serialize itself to JSON - not a clean solution, </li>
<li> we can't do the same for classes from third party libraries. We just can't make them extend traits of our preference. </li>
</ul>
{: .text-justify}

Let's think of something better.
How about sticking to the Single Responsibility Principle and pushing serialization logic out of the class?

{% highlight scala %}
trait JsonSerializer[T] {
  def toJson(t: T): Json
}

object ColorJsonSerializer extends JsonSerializer[Color] {
  override def toJson(c: Color): Json =
    s"{'name': ${c.name}, 'red': ${c.red}, 'green': ${c.green}, 'blue': ${c.blue}}"
}
{% endhighlight %}

Now we call:
{% highlight scala %}
val yellow = Color("yellow", 255, 255, 0)
val json: Json = ColorJsonSerializer.toJson(yellow)
{% endhighlight %}
to get a JSON representation of yellow.
That's better than the previous solution, but we can pimp it using some implicit magic.
First let's make **JsonSerializer.toJson** method look like it's a method of **Color** class.
To achieve that, let's add an implicit wrapper over **Color**.
{: .text-justify}

{% highlight scala %}
implicit class JsonSerializerWrapper[T](t: T) {
  def toJson(serializer: JsonSerializer[T]): Json = serializer.toJson(t)
}
{% endhighlight %}

Having this we can serialize **Color** instance:
{% highlight scala %}
val json: Json = yellow.toJson(ColorJsonSerializer)
{% endhighlight %}

Now let's add two more implicit keywords to our code. First will go here:
{% highlight scala %}
implicit object ColorJsonSerializer extends JsonSerializer[Color] {
{% endhighlight %}
and the second here:
{% highlight scala %}
def toJson(implicit serializer: JsonSerializer[T]): Json = serializer.toJson(t)
{% endhighlight %}

By making serializer argument implicit and having implicit serializer for **Color** in scope, we just call:
{% highlight scala %}
val json: Json = color.toJson
{% endhighlight %}

We achieved something great.
We built a mechanism that allows us to add method to existing classes.
**JsonSerializer[T]** is a type class.
This concept of enriching APIs is called type class pattern.
{: .text-justify}

This example was heavily inspired by a piece of code you can find in [a tutorial for JSON library spray-json][spray-type-classes].
To deepen the understanding of type classes let's study an example of how smart people behind spray-json use that pattern.
Here's a piece of code from spray-json tutorial enriched with my comments:
{: .text-justify}
{% highlight scala %}
import spray.json.{JsArray, JsNumber, JsString, JsValue, RootJsonFormat}

case class Color(name: String, red: Int, green: Int, blue: Int)

implicit object ColorJsonFormat extends RootJsonFormat[Color] {
  def write(c: Color) = JsArray(JsString(c.name), JsNumber(c.red), JsNumber(c.green), JsNumber(c.blue))

  def read(value: JsValue) = value match {
    case JsArray(Vector(JsString(name), JsNumber(red), JsNumber(green), JsNumber(blue))) =>
      new Color(name, red.toInt, green.toInt, blue.toInt)
    case _ => deserializerError("Color expected")
   }
}

import spray.json._ // check the next listing to what is imported from this package object

val json = Color("CadetBlue", 95, 158, 160).toJson //the same pattern we used in our JSON serialization
{% endhighlight %}

That's an excerpt from **spray.json** package object:
{% highlight scala %}
trait RootJsonFormat[T] extends JsonFormat[T] with RootJsonReader[T] with RootJsonWriter[T]

trait RootJsonWriter[T] extends JsonWriter[T] // an analogue to our JsonSerializer[T]

implicit def pimpAny[T](any: T) = new PimpedAny(any) // implicit conversion happens here

private[json] class PimpedAny[T](any: T) { // an analogue to our JsonSerializerWrapper[T]
    def toJson(implicit writer: JsonWriter[T]): JsValue = writer.write(any)
}
{% endhighlight %}

[spray-type-classes]: https://github.com/spray/spray-json
