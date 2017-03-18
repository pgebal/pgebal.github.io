---
layout: post
title: "A short one: post a json with akka HTTP client-side API"
date: 2017-03-16 14:00:00 +0100
categories: articles
tags: [scala, functional programming, json, akka, akka http]
excerpt:
comments: true
share: true
---

As a newcomer to akka HTTP it took me a while to figure out how to make a post request with json body using request-level akka HTTP client-side API.
I wrote this article, so you don't have to go the same path that I went.
{: .text-justify}

Imagine you start apps using a service that provides a REST API over HTTP.
You do it by posting jsons on /apps resource. Example json:
{: .text-justify}
{% highlight json %}
{
  "jarUri": "file:///home/gebal/gebal/example.jar",
  "className": "com.pawelgebal.Main",
  "args": []
}
{% endhighlight %}

Let's model it as an instance of *Application* class.
{% highlight scala %}
final case class Application(jarUri: String, className: String, args: List[String])
{% endhighlight %}

To turn *Application* instance into a json we'll use spray-json library.
Add this to your build.sbt:
{: .text-justify}
{% highlight sbt %}
val akkaHttpVersion = "10.0.4"
libraryDependencies += "com.typesafe.akka" %% "akka-http" % akkaHttpVersion
libraryDependencies += "com.typesafe.akka" %% "akka-http-spray-json" % akkaHttpVersion
libraryDependencies += "io.spray" %% "spray-json" % "1.3.3"
{% endhighlight %}

*SprayJsonSupport* trait and *akka.http.scaladsl.marshalling.Marshal* will help us to turn *Application* instance into a *RequestEntity* with a json body.
Then we'll construct a request and post it via akka HTTP client.
We are going to need a *Marshaller[App, RequestEntity]*.
We'll build it in two steps.
Let's start with a trait that provides *RootJsonWriter* for *Application*.
{: .text-justify}

{% highlight scala %}
package com.example

trait ApplicationProtocol extends DefaultJsonProtocol {
  implicit val appFormat = jsonFormat3(Application)
}
{% endhighlight %}
This writer transforms an instance of *Application* into a *JsValue*.
Let's use *SprayJsonSupport* trait to implicitly turn *RootJsonWriters* into *EntityMarshallers*.
{: .text-justify}
{% highlight scala %}
package com.example

import akka.actor.ActorSystem
import akka.http.scaladsl.Http
import akka.http.scaladsl.marshallers.sprayjson.SprayJsonSupport
import akka.http.scaladsl.marshalling.Marshal
import akka.http.scaladsl.model._
import akka.stream.ActorMaterializer

import scala.concurrent.Future
import scala.util.{Failure, Success}

object ApplicationPoster extends App with ApplicationProtocol with SprayJsonSupport {

  implicit val system = ActorSystem()
  implicit val ec = system.dispatcher
  implicit val materializer = ActorMaterializer()

  val http = Http(system)

  def postApplication(app: Application): Future[HttpResponse] =
    Marshal(app).to[RequestEntity] flatMap { entity =>
      val request = HttpRequest(method = HttpMethods.POST, uri = "http://localhost:8080/apps", entity = entity)
      http.singleRequest(request)
    }

  val application = Application("file:///home/gebal/gebal/example.jar", "com.pawelgebal.Main", Nil)

  postApplication(application) onComplete {
    case Failure(ex) => System.out.println(s"Failed to post $application, reason: $ex")
    case Success(response) => System.out.println(s"Server responded with $response")
  }
}
{% endhighlight %}

To see that our app works and produces a request with a content type *application/json*,
run this before you start *ApplicationPoster*:
{: .text-justify}
{% highlight scala %}
package com.example

import akka.http.scaladsl.model.{HttpEntity, StatusCodes}
import akka.http.scaladsl.server.{HttpApp, Route}

object Server extends HttpApp with App {

  override protected def route: Route =
    path("apps") {
      post {
        entity(as[HttpEntity]) { entity =>
          System.out.println(entity.getContentType())
          complete(StatusCodes.Created)
        }
      }
    }

  startServer("localhost", 8080)
}
{% endhighlight %}
That's all folks!
