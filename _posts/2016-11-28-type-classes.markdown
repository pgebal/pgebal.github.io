---
layout: post
title:  "Quality example of type class pattern"
date:   2016-11-28 20:00:00 +0100
categories: scala update
tags: scala functional-programming spray-json
excerpt: An explanation of type class pattern.
---
Let's have a class that we want to serialize to JSON:
{% highlight scala %}
case class Color(name: String, red: Int, green: Int, blue: Int)
{% endhighlight %}

If we would go Java style, we might have come up with something like this:
{% highlight scala %}
type Json = String

trait SerializableToJson {
  def toJson: Json
}

case class Color(name: String, red: Int, green: Int, blue: Int)
 extends SerializableToJson {
  override def toJson =
    s"{'name': '${name}', 'red': ${red}, 'green': ${green}, 'blue': ${blue}}"
}
{% endhighlight %}

Pros:
 * it works.

Cons:
 * Color knows how to serialize itself to JSON - not a clean solution
 * we can't do the same for classes from third party libraries. We just can't make them extend traits of our preference.

Let's think of something better.
How about sticking to the Single Responsibility Principle and pushing serializing logic out of the class?

{% highlight scala %}
trait JsonSerializer[T] {
  def toJson(t: T): Json
}

object ColorJsonSerializer extends JsonSerializer[Color] {
  override def toJson(c: Color): Json =
    s"{'name': ${c.name}, 'red': ${c.red}, 'green': ${c.green}, 'blue': ${c.blue}}"
}
{% endhighlight %}

Now we call
{% highlight scala %}
val json: Json = ColorJsonSerializer.toJson(color)
{% endhighlight %}
to get a JSON representation of color.
That's better then the previous solution, but we can pimp it using some implicit magic.
First let's make JsonSerializer.toJson method look like it's a method of Color class. To achieve that,
let's add an implicit wrapper over Color.

{% highlight scala %}
implicit class JsonSerializerWrapper[T](t: T) {
  def toJson(serializer: JsonSerializer[T]): Json = serializer.toJson(t)
}
{% endhighlight %}

Having this we can serialize Color instance:
{% highlight scala %}
val json: Json = color.toJson(ColorJsonSerializer)
{% endhighlight %}

Now let's add two more implicit keyword to our code. First will go here:
{% highlight scala %}
implicit object ColorJsonSerializer extends JsonSerializer[Color] {
{% endhighlight %}
and second here:
{% highlight scala %}
def toJson(implicit serializer: JsonSerializer[T]): Json = serializer.toJson(t)
{% endhighlight %}

By making serializer argument implicit and having implicit serializer for Color in scope, we can just call:
{% highlight scala %}
val json: Json = color.toJson
{% endhighlight %}

We achieved something great.
We built a mechanism that allows us to add method to existing classes.
JsonSerializer[T] is a type class.
This concept is called type class pattern.

This example was heavily inspired by a piece of code you can find in [tutorial for a JSON library spray-json][spray-type-classes].
To deepen the understanding of type classes let's study an example of how the smart people behind spray-json use type classes.


[spray-type-classes]: https://github.com/spray/spray-json
