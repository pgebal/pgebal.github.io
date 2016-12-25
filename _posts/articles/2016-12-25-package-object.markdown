---
layout: post
title: "A short one: What is package object for?"
date: 2016-12-25 14:00:00 +0100
categories: articles
tags: [scala, functional programming, package object]
excerpt: To save you lines of code.
comments: true
share: true
---

If you want to use some code across a package without explicitly importing it put that code into a package object.
Look at the content of *com/pawelgebal/pobj/user/package.scala*
{: .text-justify}
{% highlight scala %}
package com.pawelgebal.pobj.user

object `package` {
  type Login = String
  type Email = String
  type RegistrationTimestamp = Long
}
{% endhighlight %}

Now in other files in **com.pawelgebal.pobj.user** package you can use that types without importing them.
{: .text-justify}
{% highlight scala %}
package com.pawelgebal.pobj.user

final case class User(login: Login, email: Email)
{% endhighlight %}

If you need to use code from a package object in other package just import package name followed by '._'.
Keep in mind that imports all definitions that are publicly visible in the package object.
{: .text-justify}
{% highlight scala %}
package com.pawelgebal.pobj.service

import com.pawelgebal.pobj.user._

trait UserService {

  def registedAt(login: Login): RegistrationTimestamp

}
{% endhighlight %}

You can have one package object per package.
I encourage you to take a look on package objects for packages **scala** and **scala.concurrent.duration**.
You'll find implicit conversions, type aliases and you'll get the idea how pros use package objects.
{: .text-justify}

When studying **scala.concurrent.duration** package object you must have noticed it uses a different syntax.

{% highlight scala %}
package scala.concurrent // scala.concurrent instead of scala.concurrent.duration

package object duration extends scala.AnyRef { // package object duration instead of object `package`
  ...
}
{% endhighlight %}

That's a syntax sugar for creating a package object.
At first it may look a bit unintuitive but use this syntax to be in line with majority of Scala developers.
{: .text-justify}
