---
layout: post
title: "A short one: Reactive streaming from HDFS"
date: 2017-08-31 21:00:00 +0100
categories: articles
tags: [scala, Akka Streams, reactive, HDFS, akka]
excerpt: Learn how to stream text files from HDFS.
comments: true
share: true
---

If you want to stream a big text file from HDFS, take a reactive approach and read this post to learn how to do it.
{: .text-justify}

A great way of going reactive in Scala is using Akka Streams.
If you know nothing about Akka Streams go read [this][akka-streams-intro].
Akka Streams provides *StreamConverters.fromInputStream* method that creates a *Source* from a file content.
Emitted elements are *chunkSize* bytes (default value of 8192), except for the final element that can be smaller.
Knowing that, let's get an *InputStream* to a file stored on HDFS:
{: .text-justify}
{% highlight scala %}
import org.apache.hadoop.fs.{FileSystem, Path}

// sets HDFS host
val configuration = new Configuration()
configuration.set("fs.defaultFS", host)

val hdfs = FileSystem.get(configuration)
val textStream: InputStream = hdfs.open(new Path(filePath))
{% endhighlight %}

Once we have the stream we need to pass it to *StreamConverters.fromInputStream*.
That's how we do it:
{: .text-justify}
{% highlight scala %}
import akka.stream.scaladsl.{Framing, Sink, StreamConverters}

// assume there is an implicit Materializer in scope.
StreamConverters
  .fromInputStream(() => textStream)
  .via(Framing.delimiter(delimiter = "\n", maximumFrameLength = 500, allowTruncation = true))
  .map(_.utf8String)
  .runWith(Sink.foreach(println))    
{% endhighlight %}
After creating the *Source*, we connect it to a *Flow* using *.via* method.
*Framing.delimiter* flow buffers read bytes and chunks them into frames using a specific byte-sequence to mark frame boundaries.
In our case that special sequence is the end of line character.
If there are no delimiters in a block of *maximumFrameLength*, stream fails.
*Framing.delimiter* flow produces *ByteStrings*, which need to be mapped to utf-8 strings to make them human readable.
Finally we connect our *Flow* to a *Sink* that prints each input element and run it using *.runWith*.
That's how Akka Streams rolls with your big files.
{: .text-justify}

Check out a full example on my [github][hdfs-stream-github]


[akka-streams-intro]: http://doc.akka.io/docs/akka/2.5.3/scala/stream/stream-introduction.html
[hdfs-stream-github]: https://github.com/pgebal/hdfs-stream
