---
title: "Tailing the MongoDB Replica Set Oplog with Scala and Akka Streams"
layout: post
date: 2016-03-01 17:00
tag:
- scala
- akka
- streams
- mongodb
- scala mongo driver
blog: true
---

## Introduction:
In this post I'll try to explain how to tail MongoDB oplog using [MongoDB Scala Driver](http://mongodb.github.io/mongo-scala-driver/) and [Akka Streams](http://doc.akka.io/docs/akka/2.4.2/scala/stream/index.html).

Examples shouldn't be considered and used as production ready.

All of us know the Unix command "tail -f", a tailable cursors have pretty much the same concept. MongoDB provides a good functionality for tailing cursors and it doesn't require any extra library or toolbox, when it comes to oplog nothing changes here, it's the same collection as any other.

If you want to know more about oplog and tailable cursors you can find information on MongoDB documentation hub:

- [Replica Set Oplog](https://docs.mongodb.org/manual/core/replica-set-oplog/)
- [Create Tailable Cursor](https://docs.mongodb.org/manual/tutorial/create-tailable-cursor/)

The full project could be found [here](https://github.com/htimur/mongo_oplog_akka_streams)

## Libraries and tools:

- [Scala v2.11.7](http://www.scala-lang.org/documentation/getting-started.html)
- [Mongo Scala Driver v1.1.0](http://mongodb.github.io/mongo-scala-driver/1.1/getting-started/)
- [Akka Stream module v2.4.2](http://doc.akka.io/docs/akka/2.4.2/scala/stream/index.html)

Here is the build file:
{% gist htimur/9776f9100363ff9cdf73 %}

You will also need MongoDB Replica Set, I would recommend to use official [mongo docker image](https://hub.docker.com/_/mongo/).

#### MongoDB Scala Driver and Akka Streams

Unfortunately there is no built in functionality to integrate Akka Strams and MongoDB driver, but Akka Streams have a good integration with [Reactive Streams](http://www.reactive-streams.org/) as well as recently released new official asynchronous scala driver from MongoDB. It is using `Observable` model, which can be converted to Reactive Streams `Publisher` in few lines of code, also they already provide an [implicit based conversion example](https://github.com/mongodb/mongo-scala-driver/blob/master/examples/src/test/scala/rxStreams/Implicits.scala) which we will be using as integration point between these two libraries.

## MongoDB oplog tailing query

Now we should define the query, assuming that we already have an esteblished connection. Here is the example:
{% gist htimur/68f2639d98269c1026a0 %}

As you can see we are defining a tailable cursor with no timeout, also the `op` field of oplog document which is defining the operation, should be a CRUD operation `i/d/u`.

## Defining the stream Source

Defining the source is pretty simple, from oplog we will receive `Document` objects that will be the type of our source.
{% highlight scala %}
val source: Source[Document, NotUsed] = _
{% endhighlight %}

So we've got a `FindObservable[Document]` from MongoDB oplog query and the source `Source[Document, NotUsed]` how do we convert from one to another?

Here where the implicit magic happens, the `Source` companion object have a method

{% highlight scala %}
def fromPublisher[T](publisher: Publisher[T]): Source[T, NotUsed]
{% endhighlight %}

which returns a source from Reactive Streams `Publisher` also we have got an implicit conversion from MongoDB `Observable` to `Publisher`. So lets glue everything together:

{% gist htimur/6eb4c041f1a8845f5205 %}

That was easy, isn't it? =)

#### What next?

Anything you can imagine, for now I'll just print the documents to `STDOUT`

{% highlight scala %}
source.runForeach(println)
{% endhighlight %}

this will print all CRUD operations on repset from begining to end and will be waiting for any new operations.

You can make more specific query and define the database and collection names you are interested in, also define the time from which you want the documents. I'll leave this up to you.

## Summary

As you can see tailing the MongoDB oplog is really easy, especially on Replica Set. There will be more pitfalls if it's a MongoDB Sharded Cluster, which I'll try to cover on my next post.

Of course this post doesn't cover every aspect of the topic, for example failure handling, delivery guarantees and etc.  That's something which can be implemented in different ways and isn't the target for current post.
