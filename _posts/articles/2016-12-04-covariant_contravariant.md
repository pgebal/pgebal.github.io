---
layout: post
title:  "A short one: Covariant type in a contravariant position"
date:   2016-12-04 12:00:00 +0100
categories: articles
tags: [scala, functional programming, covariance, contravariance]
excerpt: It will never happen to you after reading this.
comments: true
share: true
---
There is an exercise in the great [Functional Programming in Scala][functional-programming-in-scala-affiliate-link] book, that requires you to
implement **Option** trait and it's **getOrElse** method.
My first try on it was:
{: .text-justify}
{% highlight scala %}
sealed trait Option[+A] {

  def getOrElse(default: => A): A = this match {
    case Some(a) => a
    case None => default
  }
}

case class Some[+A](value: A) extends Option[A]
case object None extends Option[Nothing]
{% endhighlight %}

IntelliJ Idea was quick to tell me that it's not a right solution because:
{% highlight scala %}
Covariant type A in contravariant position in type A of value default.
{% endhighlight %}

I looked to the book and noticed that they suggest to use the following definition for **getOrElse**:
{% highlight scala %}
def getOrElse[B >: A](default: => B): B = this match {
  case Some(a) => a
  case None => default
}
{% endhighlight %}

At first I didn't know why the solution required to use type parameter **B** that is a supertype of **A**.
It turned out to be quite simple.
Notice that **A** is a covariant type parameter for **Option**.
That means for every **A** that is a subtype of **B**, **Option[A]** is a subtype of **Option[B]**.
So whenever we use expression of type **Option[A]**, we have to be able to substitute it with expression of type **Option[B]**.
Study this:
{: .text-justify}

{% highlight scala %}
class B
class A extends B

val b: Option[B] = None
val a: Option[A] = None

val default: B = b.getOrElse(new B)
{% endhighlight %}
In the last line we could use **a** instead of **b** and our code would stay valid. That's thanks to <nobr><b>[B >: A]</b></nobr> bound in **getOrElse** method definition. If it wasn't for that **a.getOrElse** would not accept an argument of type **B**. Keep this example in mind and embrace [Liskov substitution principle][liskov-substitution-principle] when implementing your own classes with type parameters.
{: .text-justify}

[functional-programming-in-scala-affiliate-link]: https://www.amazon.com/gp/product/1617290653/ref=as_li_tl?ie=UTF8&camp=1789&creative=9325&creativeASIN=1617290653&linkCode=as2&tag=pawelgebal-20&linkId=f3ec949cadbc0ff936a5aae1dcc51c0a
[liskov-substitution-principle]:https://en.wikipedia.org/wiki/Liskov_substitution_principle
