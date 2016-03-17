---
title: "Tailing the MongoDB Oplog on Sharded Clusters with Scala and Akka Streams"
layout: post
date: 2016-03-16 21:00
tag:
- scala
- akka
- streams
- mongodb
- scala mongo driver
blog: true
---

## Introduction
The post is a continuation of a previously published post [Tailing the MongoDB Replica Set Oplog with Scala and Akka Streams](http://khamrakulov.de/tailing_the_mongodb_replica_set_oplog_with_scala_and_akka_streams/).

As it was discussed previously, tailing the MongoDB Oplog on Sharded Cluster have some pitfalls compared to the Replica Set. This post will try to cover some aspects of this topic and little bit more.

There are 2 really good article fully covering the topic of Tailing the MongoDB Oplog on Sharded Clusters from MongoDB team. You can find them here:

- [Tailing the MongoDB Oplog on Sharded Clusters](https://www.mongodb.com/blog/post/tailing-mongodb-oplog-sharded-clusters)
- [Pitfalls and Workarounds for Tailing the Oplog on a MongoDB Sharded Cluster](https://www.mongodb.com/blog/post/pitfalls-and-workarounds-for-tailing-the-oplog-on-a-mongodb-sharded-cluster)

You can also find more information about the MongoDB Sharded Cluster in [documentation hub](https://docs.mongodb.org/manual/core/sharding-introduction/).

The project built as example is available in [github](https://github.com/htimur/mongo_oplog_akka_streams).

The examples provided in this post shouldn't be considered and used as production ready.

## Libraries and tools

Nothing changes here, we will use the same dependencies as in previous post, but now we will need MongoDB Sharded Cluster instead of Replica Set.

## MongoDB Sharded Cluster

From MongoDB documentation:

>**Sharding**, or horizontal scaling, by contrast, divides the data set and distributes the data over multiple servers, or shards. Each shard is an independent database, and collectively, the shards make up a single logical database.

![Sharded Collection]({{ site.url }}/assets/images/sharded-collection.png)

In production architecture each shard is represented by Replica Set:

![Sharded Cluster Architecture]({{ site.url }}/assets/images/sharded-cluster-production-architecture.png)

## MongoDB internal operations

Due to distribution of data on multiple nodes, MongoDB have cluster-internal operations  [`balancing`](https://docs.mongodb.org/manual/tutorial/manage-sharded-cluster-balancer/), which are reflected in `oplog`. These operations have extra field `fromMigrate`, since we are not interested in these operations, we will update our `oplog` query to exclude them.

{% gist htimur/a87dce28668a82000fcf %}

## Retrieving Shard Information

As you might have guest already, to tail the Oplog on Sharded Cluster we should tail the Oplogs of each shard (Replica Sets).

To do so we can query the `config` database to get the list of available shards. The documents in the collection look something like:

{% gist htimur/279fb96b457b6c0f1a69 %}

I preferrer to use a `case` classes instead of `Document` objects, so I'll define it:

{% highlight scala %}
case class Shard(name: String, uri: String)
{% endhighlight %}

and a method to parse `Document` to `Shard`:

{% gist htimur/de8b340dd92322bb6096 %}
and now we can query the collection:

{% gist htimur/95cba05b006fc21e40c8 %}

In the end we have the list of all shards in our MongoDB Sharded Cluster.

## Defining the Source for each shard

To define the `Source`, we can simply iterate over the list of shards and use the same method, as in previous article, to define it per shard.

{% gist htimur/11f79e2ba836c42e3da5 %}

### Merging Sources

We could process each `Source` separately, but of course it's much easier and more comfortable to work with them as with single `Source`. To do so, we should merge them.

In Akka Streams there are multiple `Fan-in` operations:

>
* **Merge[In]** – (N inputs , 1 output) picks randomly from inputs pushing them one by one to its output
* **MergePreferred[In]** – like Merge but if elements are available on preferred port, it picks from it, otherwise randomly from others
* **ZipWith[A,B,...,Out]** – (N inputs, 1 output) which takes a function of N inputs that given a value for each input emits 1 output element
* **Zip[A,B]** – (2 inputs, 1 output) is a ZipWith specialised to zipping input streams of A and B into an (A,B) tuple stream
* **Concat[A]** – (2 inputs, 1 output) concatenates two streams (first consume one, then the second one)

We will use simplified API for [`Merge`](http://doc.akka.io/docs/akka/2.4.2/scala/stream/stream-graphs.html#combining-sources-and-sinks-with-simplified-api) and then output the stream to `STDOUT`:

{% gist htimur/d6359c8ad395bfeb257d %}

## Error Handling - Failovers and rollbacks

For error handling Akka Streams use [`Supervision Strategies`](http://doc.akka.io/docs/akka/2.4.2/scala/stream/stream-error.html). There are 3 ways to handle exceptions:

>
* **Stop** - The stream is completed with failure.
* **Resume** - The element is dropped and the stream continues.
* **Restart** - The element is dropped and the stream continues after restarting the stage. Restarting a stage means that any accumulated state is cleared. This is typically performed by creating a new instance of the stage.

But unfortunately it's not applicable to `ActorPublisher` source and `ActorSubscriber` sink components, so in case of `failovers` and `rollbacks` our `Source` will not be able to recover properly.

There is already an issue opened in github, [#16916](https://github.com/akka/akka/issues/16916) I hope it'll be fixed soon.

As an alternative you could also consider a totally different approach suggested in the article [Pitfalls and Workarounds for Tailing the Oplog on a MongoDB Sharded Cluster](https://www.mongodb.com/blog/post/pitfalls-and-workarounds-for-tailing-the-oplog-on-a-mongodb-sharded-cluster).

>Finally a completely different approach would be to tail the oplogs of a majority or even all nodes in a replica set. Since the pair of the `ts & h` fields uniquely identifies each transaction, it is possible to easily merge the results from each oplog on the application side so that the "output" of the tailing thread are the events that have been returned by at least a majority of MongoDB nodes. In this approach you don't need to care about whether a node is a primary or secondary, you just tail the oplog of all of them and all events that are returned by a majority of oplogs are considered valid. If you receive events that do not exist in a majority of the oplogs, such events are skipped and discarded.

## Summary

As you can see, there are some aspects which are pretty easy to handle with Akka Streams and those which are not. In general, I have mixed impression about the library. It have a good ideas, leveraging the Akka Actors and moving it to the next level, but it feels raw. Personally I'll stick with Akka Actors for now.

We didn't cover the `Updates to orphan documents in a sharded cluster`, as in my case I'm interested in all operations and consider them as idempotent per `_id` field, so it doesn't hurt.
