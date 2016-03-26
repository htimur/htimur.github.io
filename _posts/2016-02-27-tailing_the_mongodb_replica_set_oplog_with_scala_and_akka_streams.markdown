---
title: "Tailing the MongoDB Replica Set Oplog with Scala and Akka Streams"
layout: post
date: 2016-03-01 17:00
tag:
- scala
- akka
- streams
- reactive programming
- mongodb
- scala mongo driver
blog: true
---

## Introduction
In this post I'll try to explain how to tail MongoDB Oplog using [MongoDB Scala Driver](http://mongodb.github.io/mongo-scala-driver/1.1/getting-started/) and [Akka Streams](http://doc.akka.io/docs/akka/2.4.2/scala/stream/index.html).

The examples provided in this post shouldn't be considered and used as production ready.

All of us know the Unix command `tail -f`, a tailable cursors have pretty much the same concept. MongoDB provides a good functionality for tailing cursors and it doesn't require any extra library or toolbox. When it comes to Oplog nothing changes here, it's the same collection as any other.

If you want to know more about Oplog and tailable cursors you can find information on MongoDB documentation hub:

- [Replica Set Oplog](https://docs.mongodb.org/manual/core/replica-set-oplog/)
- [Create Tailable Cursor](https://docs.mongodb.org/manual/tutorial/create-tailable-cursor/)

The full project could be found [here](https://github.com/htimur/mongo_oplog_akka_streams)

## Libraries and tools

- [Scala v2.11.7](http://www.scala-lang.org/documentation/getting-started.html)
- [Mongo Scala Driver v1.1.0](http://mongodb.github.io/mongo-scala-driver/1.1/getting-started/)
- [Akka Streams module v2.4.2](http://doc.akka.io/docs/akka/2.4.2/scala/stream/index.html)

Here is the build file:
{% gist htimur/9776f9100363ff9cdf73 %}

You will also need MongoDB Replica Set, I would recommend to use official [mongo docker image](https://hub.docker.com/_/mongo/).

## MongoDB Oplog tailing query

Assuming that we already have an established connection, lets define the query. Here is an example:
{% gist htimur/68f2639d98269c1026a0 %}

As you can see we are defining a tailable cursor with no timeout, also the `op` field of Oplog document which is defining the operation, should be a [CRUD](https://en.wikipedia.org/wiki/Create,_read,_update_and_delete) operation `i/d/u`.

## Small Introduction to Akka Streams terminology

From [documentation](http://doc.akka.io/docs/akka/2.4.2/scala/stream/index.html):

>Linear processing pipelines can be expressed in Akka Streams using the following core abstractions:

> **Source** - A processing stage with exactly one output, emitting data elements whenever downstream processing stages are ready to receive them.

> **Sink** - A processing stage with exactly one input, requesting and accepting data elements possibly slowing down the upstream producer of elements

> **Flow** - A processing stage which has exactly one input and output, which connects its up- and downstreams by transforming the data elements flowing through it.

> **RunnableGraph** - A Flow that has both ends "attached" to a Source and Sink respectively, and is ready to be run().

In our case we will be using only `Source` and `Sink`, as we will be only tailing cursor without any transformations.

For more information please check the [documentation](http://doc.akka.io/docs/akka/2.4.2/scala/stream/index.html).

#### MongoDB Scala Driver and Akka Streams

Unfortunately, there is no built-in functionality to integrate Akka Streams and MongoDB driver, but Akka Streams have a good integration with [Reactive Streams](http://www.reactive-streams.org/) as well as the recently released new official asynchronous scala driver from MongoDB. It is using the `Observable` model, which can be converted to Reactive Streams `Publisher` in a few lines of code, also the MongoDB team provide an [implicit based conversion example](https://github.com/mongodb/mongo-scala-driver/blob/master/examples/src/test/scala/rxStreams/Implicits.scala) which we will be using as an integration point between these two libraries.

## Defining the stream Source

Defining the source is pretty simple, from Oplog we will receive `Document` objects that will be the type of our source.
{% highlight scala %}
val source: Source[Document, NotUsed] = _
{% endhighlight %}

So we've got a `FindObservable[Document]` from MongoDB Oplog query and the source `Source[Document, NotUsed]` how do we convert from one to another?

Here's where the implicit magic happens, the `Source` companion object has a method

{% highlight scala %}
def fromPublisher[T](publisher: Publisher[T]): Source[T, NotUsed]
{% endhighlight %}

which returns a source from Reactive Streams `Publisher`, also we have got an implicit conversion from MongoDB `Observable` to `Publisher`. So lets glue everything together:

{% gist htimur/6eb4c041f1a8845f5205 %}

That was easy, isn't it?

#### What next?

Anything you can imagine, for now I'll just print the documents to `STDOUT`

{% highlight scala %}
source.runWith(Sink.foreach(println))
{% endhighlight %}

or use shortcut:

{% highlight scala %}
source.runForeach(println)
{% endhighlight %}

this will print all CRUD operations on repset from begining to end and will be waiting for any new operations.

You can make more specific query and define the database and collection names you are interested in, also define the time from which you want the documents. I'll leave this up to you.

#### Why?

You may wonder why do we need to do all this conversions from `Observable` to `Publisher` and then to the `Source`, when we could simply stick to the Reactive Streams `Publisher` or even use `Observables`.

Well `Observables` model is quite similar to Reactive Streams API, but they just define a common mechanism of how to move data across an asynchronous boundary without losses, buffering or resource exhaustion, when Akka Streams API is focusing on formulation of transformations on data streams.

So if you are interested only in moving data from Oplog somewhere else you could use the `Observables` model from MongoDB driver, but as soon as there is a need in transformations then Akka Streams is the choice.

## Summary

As you can see, tailing the MongoDB Oplog is really easy, especially on Replica Set. There will be more pitfalls if it's a MongoDB Sharded Cluster, which I'll try to cover on my next post.

Of course this post doesn't cover every aspect of the topic, for example failure handling, delivery guarantees and etc.  That's something which can be implemented in different ways and isn't the target for current post.
