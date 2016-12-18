---
layout: post
title: "A short one: foldRight via foldLeft"
date: 2016-12-18 14:00:00 +0100
categories: scala update
tags: scala functional-programming functional programming induction
excerpt: Solution to the puzzle.
---
<script type="text/javascript"
    src="http://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML">
</script>

There is a great exercise in [Functional programming in Scala][functional-programming-in-scala-affiliate-link] that forces you to understand how **List**'s **foldRight** and **foldLeft** work.
It requires you to implement **foldRight** in terms of **foldLeft**.
Following piece of code introduces data structures and methods that we will deal with in this post:
{% highlight scala %}
sealed trait List[+A]

case object Nil extends List[Nothing]

case class Cons[+A](head: A, tail: List[A]) extends List[A]

object List {

  def foldRight[A,B](as: List[A], z: B)(f: (A, B) => B): B =
    as match {
      case Nil => z
      case Cons(x, xs) => f(x, foldRight(xs, z)(f))
  }

  @annotation.tailrec
  def foldLeft[A,B](as: List[A], z: B)(g: (B, A) => B): B =
    as match {
      case Nil => z
      case Cons(h,t) => foldLeft(t, g(z,h))(g)
  }

  def foldRightViaFoldLeft[A, B](as: List[A], z: B)(f: (A, B) => B): B = ???
}
{% endhighlight %}

Why should we care about expressing **foldRight** via **foldLeft**?
The latter is tail recursive so we won't get into stack overflow when we use **foldRightViaFoldLeft** for long lists.
{: .text-justify}

### Understanding what folds do ###

From now on, outside Scala code blocks we will represent lists of length n as \\( [a_1, a_2, ..., a_n] \\).
{: .text-justify}

Having a list \\( [a_1, a_2, a_3, ..., a_n] \\), function \\( g: (B, A) => B \\) and element \\( z \in B \\) **foldLeft** returns the value of:
\\( g(...g(g(g(z, a_1), a_2),a_3), ...), a_n) \\)

**foldRight** for a list \\( [a_1, a_2, a_3, ..., a_{n-1}, a_n] \\), function \\( f: (A, B) => B \\) and element \\( z \in B \\) returns the value of expression:
\\( f(a_1, f(a_2, f(....,f(a_{n-1}, f(a_n, z))...))) \\)

### Understanding what we need to do ###

We need to define arguments for **foldLeft** in terms of **foldRight** arguments **as**, **z** and **f**.

### Solution ###

My first idea that worked was:
{% highlight scala %}
def reverse[A](as: List[A]): List[A] =
        foldLeft(as, Nil: List[A])((acc, a) => Cons(a, acc))

def foldRightViaFoldLeft[A, B](as: List[A], z: B)(f: (A, B) => B): B =
        foldLeft(reverse(as), z)((b, a) => f(a, b))
{% endhighlight %}
In this case arugment **g** for **foldLeft** is **((b, a) => f(a, b))**
We won't get into a strict proof here, but let's understand why **foldRightViaFoldLeft** returns the value of expression \\( f(a_1, f(a_2, f(....,f(a_{n-1}, f(a_n, b))...))) \\).
Let's start with list of size 1:
{: .text-justify}

Have in mind that for every \\( a \in A, b\in B, g(b, a) = f(a, b) \\):

$$
reverse(as) = reverse([a_1]) = [a_1] \\
foldLeft([a_1], z)(g) = g(z, a_1) = f(a_1, z) = foldRight([a_1], z)(f)
$$

No let's try this with list of size 2:

$$
reverse(as) = reverse([a_1, a_2]) = [a_2, a_1] \\
foldLeft([a_2, a_1], z)(g) = g(g(z, a_2), a_1) = g(f(a_2, z), a_1) = f(a_1, f(a_2, z)) = \\
= foldRight([a_1, a_2], z)(g)
$$

Let's assume that the solution works for lists of fixed length n:

$$
foldLeft([a_n, a_{n-1}, ..., a_1], z)(g) = foldRight([a_1, a_2, ..., a_n], z)(f)
$$

From that we'll conclude that it works for list of length n + 1.
From the definition of **foldLeft** we have that:

$$
foldLeft([a_{n+1}, a_n, ..., a_1], z)(g) = foldLeft([a_n, a_{n-1}, ..., a_1], g(z, a_{n+1}))(g)
$$

From the assumption we know that:

$$
foldLeft([a_n, a_{n-1}, ..., a_1], g(z, a_{n+1}))(g) = foldRight([a_1, a_2, ..., a_n], g(z, a_{n+1}))(f) = \\
= foldRight([a_1, a_2, ..., a_n], f(a_{n+1}, z))(f) = foldRight([a_1, a_2, ..., a_n, a_{n+1}], z)(f)
$$

We're done. That was [Mathematical induction][mathematical-induction].

### Solution that does not use reverse ###

After solving this, I started to look for solution that does not use reverse.
I couldn't come up with anything so I've peeked into [FPIS][functional-programming-in-scala-affiliate-link] answers and found out a tricky solution that does not use reverse.
Here it is:
{% highlight scala %}
def foldRightViaFoldLeft_1[A,B](l: List[A], z: B)(f: (A,B) => B): B =
        foldLeft(l, (b:B) => b)((g,a) => b => g(f(a,b)))(z)
{% endhighlight %}

Try to use induction and prove that **foldRightViaFoldLeft_1** is correct.


[functional-programming-in-scala-affiliate-link]: https://www.amazon.com/gp/product/1617290653/ref=as_li_tl?ie=UTF8&camp=1789&creative=9325&creativeASIN=1617290653&linkCode=as2&tag=pawelgebal-20&linkId=f3ec949cadbc0ff936a5aae1dcc51c0a
[fpis-answers]:https://github.com/fpinscala/fpinscala
[mathematical-induction]:https://en.wikipedia.org/wiki/Mathematical_induction
