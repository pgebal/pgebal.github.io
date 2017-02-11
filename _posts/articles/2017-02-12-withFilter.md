---
layout: post
title: "A short one: withFilter"
date: 2017-02-12 14:00:00 +0100
categories: articles
tags: [scala, functional programming, collections, filter, withFilter]
excerpt: Let your code be lazy with withFilter.
comments: true
share: true
---

I've been using Scala for over a year, so I thought it's finally the time to learn how for comprehensions work under the hood.
While studying this topic I bumped into *TraversableLike.withFilter* method.
I've never been using *withFilter* explicitly, which is bad, because I should have started using it a long time ago.
You should do the same and in this post you will find out why.
{: .text-justify}

Scala allows you to filter *TraversableLike* and *Option* in two ways: using *filter* and *withFilter* methods.
The difference between those two is that the latter is lazy and pushes filtering until you *map*, *flatMap* or traverse container using *foreach*.
Here are the signatures of those methods for *TraversableLike[+A, +Repr]*, as of Scala 2.11.8:
{: .text-justify}

{% highlight scala %}
def filter(p: A => Boolean): Repr
def withFilter(p: A => Boolean): FilterMonadic[A, Repr]
{% endhighlight %}

*filter* is eager and with every call returns a new container that contains only the elements satisfying predicate *p*, and *withFilter* does not traverse the container but gets you a *FilterMonadic*, which is:
{: .text-justify}
{% highlight scala %}
trait FilterMonadic[+A, +Repr] extends Any {
  def map[B, That](f: A => B)(implicit bf: CanBuildFrom[Repr, B, That]): That
  def flatMap[B, That](f: A => scala.collection.GenTraversableOnce[B])(implicit bf: CanBuildFrom[Repr, B, That]): That
  def foreach[U](f: A => U): Unit
  def withFilter(p: A => Boolean): FilterMonadic[A, Repr]
}
{% endhighlight %}

Here's an implementation of *FilterMonadic* that *TraversableLike* uses (*p* is the predicate by witch we filter):
{: .text-justify}
{% highlight scala %}
class WithFilter(p: A => Boolean) extends FilterMonadic[A, Repr] {

  def map[B, That](f: A => B)
      (implicit bf: CanBuildFrom[Repr, B, That]): That = {
    val b = bf(repr)
    for (x <- self)
      if (p(x)) b += f(x)
    b.result
  }

  def flatMap[B, That](f: A => GenTraversableOnce[B])
      (implicit bf: CanBuildFrom[Repr, B, That]): That = {
    val b = bf(repr)
    for (x <- self)
      if (p(x)) b ++= f(x).seq
    b.result
   }

   def foreach[U](f: A => U): Unit =
     for (x <- self)
       if (p(x)) f(x)

   def withFilter(q: A => Boolean): WithFilter =
     new WithFilter(x => p(x) && q(x))
}      
{% endhighlight %}
Notice that *WithFilter* wraps your *TraversableLike* and doesn't do any work until values are pulled from the container in *for* loops.

*WithFilter* does not implement *filter* method, so the following piece of code won't compile:
{: .text-justify}
{% highlight scala %}
List.range(0, 1000).withFilter(_ > 10).filter(_ < 20)
{% endhighlight %}

If you need to filter *WithFilter* instance you have to call *withFilter* method again.
Here's an example:
{: .text-justify}
{% highlight scala %}
import scala.collection.generic.FilterMonadic

val isEven: Int => Boolean = _ % 2 == 0
val isLongerThanOneDigit: Int => Boolean = _ >= 10

val evensLongerThanOneDigit: FilterMonadic[Int, List[Int]] =
  List.range(0, 1000)
    .withFilter(isEven)
    .withFilter(isLongerThanOneDigit)

evensLongerThanOneDigit.foreach(println)    
{% endhighlight %}
*evensLongerThanOneDigit* is a *WithFilter* with a predicate that is a conjunction of *isEven* and *isLongerThanOneDigit*.
The code above will print all natural numbers from range (0, 100) that are even and longer that one digit in their decimal representation.
{: .text-justify}

Now you know that if you want to efficiently *map* or *flatMap* over filtered traversable you should use *withFilter*.
