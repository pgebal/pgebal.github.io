---
layout: post
title:  "A short one: A method, a function and an eta-expansion"
date:   2016-11-20 20:11:00 +0100
categories: articles
tags: [scala, functional programming, eta expansion]
excerpt: How Scala uses eta-expansion to make your life easier.
comments: true
share: true
---
A method:
{% highlight scala %}
scala> def method(x: Int) = 2 * x
method: (x: Int)Int
{% endhighlight %}

A function:
{% highlight scala %}
scala> val function: Int => Int = 2 * _
function: Int => Int = <function1>
{% endhighlight %}

A method is not a function:
{% highlight scala %}
scala> function.toString
res2: String = <function1>

scala> method.toString
<console>:9: error: missing arguments for method method;
follow this method with `_' if you want to treat it as a partially
              applied function method.toString
{% endhighlight %}

But in the following case you can use both in the same way:
{% highlight scala %}
List(1, 2, 3).map(method)
res4: List[Int] = List(2, 4, 6)
List(1, 2, 3).map(function)
res6: List[Int] = List(2, 4, 6)
{% endhighlight %}

Let's study the signature of List.map (just the relevant part) to see why is that possible:
{% highlight scala %}
final override def map[B, That](f: scala.Function1[A, B])(implicit bf: ...) : That = ...
{% endhighlight %}
Notice that argument **f** has to be a **Function[Int, Int]**. Yet somehow we passed a method and compiler didn't throw an error at us.
How come? The answer is: there is an [implicit conversion under the hood][scala-method-conversion].

The conversion is based on a trick called *eta-expansion* or *eta-abstraction* in which you expand *lambda expression* (in our case that would be **method**) to *lambda abstraction* (**methodAsFunction**).
{% highlight scala %}
val methodAsFunction: Int => Int = a => method(a)
{% endhighlight %}
That's basically what scala does to convert a method to a function.
How does it know which method to expand when there are overloaded methods?
I'll explain it some other time.

# Hint: #
If you want to explicitly convert a method into a function use underscore after method name:
{% highlight scala %}
scala> (method _).toString
res5: String = <function1>
{% endhighlight %}
Read [this][scala-method-values] for more details.

[scala-method-conversion]: http://scala-lang.org/files/archive/spec/2.11/06-expressions.html#method-conversions
[scala-method-values]: http://scala-lang.org/files/archive/spec/2.11/06-expressions.html#method-values
