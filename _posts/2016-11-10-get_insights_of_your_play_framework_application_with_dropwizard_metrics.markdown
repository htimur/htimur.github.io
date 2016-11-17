---
title: "Get insights on your Play Framework application with Dropwizard Metrics"
layout: post
date: 2016-11-17 16:00
tag:
- scala
- play framework
- dropwizard metrics
- metrics
- monitoring
blog: true
---

## Introduction

At some point of the application development, all of us are reaching the point, when we need more insights into what is happening inside our applications or need monitoring of the application. In case of the Play Framework there is already a drop in solution provided by a great open source tool [Kamon](http://kamon.io/) and it's module [kamon-play](http://kamon.io/integrations/web-and-http-toolkits/play/).

But today we are going to talk about an alternative solution, [Drowizard Metrics](http://metrics.dropwizard.io) formerly known as `Codahale Metrics`, and its integration and usage with Play Framework.

## Integration

Well, at this point I started to look around wondering if there are any libraries that could provide integration of these two.

I found multiple libraries, but not complete though. For example:

* [metrics-scala](https://github.com/erikvanoosten/metrics-scala) - Great library, neat API and Scala support, but in terms of Play Framework, just not enough.
* [metrics-play](https://github.com/kenshoo/metrics-play) - One of the first modules that Google is trying to `satisfy you and get away`, but it's already abandoned and isn't compatible with the latest versions of the Play Framework and Metrics. There is a [fork](https://github.com/breadfan/metrics-play) though, that is updated to the latest library versions, so I decided to give it a try.

Unfortunately [metrics-play](https://github.com/breadfan/metrics-play) module has a very basic support of the environment Dropwizard Metrics provides. It should be enough if you just need basic metrics which are exposed through REST api, but I had higher requirements and ended up extending the functionality of the module and building following modules:

* [metrics-reporter-play](https://github.com/htimur/metrics-reporter-play) - Metrics reporter integration with Play Framework.
* [metrics-annotation-play](https://github.com/htimur/metrics-annotation-play) - Metrics Annotations Support for Play Framework through Guice AOP.

That's what we are going to talk about next.

#### Metrics reporter integration with Play Framework

> Metrics provides a powerful toolkit of ways to measure the behavior of critical components in your production environment.

As well as expose measured data through reporters, that's a great way of pushing the stats from your application to preferred storage backend.

At the time of writing the article, the supported reporters are:

* console - Reports metrics periodically to the console.
* graphite - Reports metrics periodically to Graphite.

But Metrics library and community also provide various reporters like `Ganglia Reporter`, `CSV Reporter`, `InfluxDB Reporter`, `ElasticSearch Reporter` and other.

Adding factories for reporters is no-brainer.

#### Metrics Annotations Support for Play Framework through Guice AOP.

By default to create and track metrics, we should invoke metric registry, create metrics and so on. Let's take a look:

{% gist htimur/492c5d73f6d13c4f38ee54babe6c7093 %}

To keep it `DRY` there are annotations that you can use, the module will create and appropriately invoke a Timer for `@Timed`, a Meter for `@Metered`, a Counter for `@Counted`, and a Gauge for `@Gauge`. `@ExceptionMetered` is also supported, this creates a Meter that measures how often a method throws exceptions.

The above example can be rewritten like:

{% gist htimur/98419a7d55fd287e7816fb94721271ac %}

or you can even annotate the class, to create metrics for all declared methods of it:

{% gist htimur/7001db60e2bbcd0cfe76b40d9f748950 %}

Of course this is supported only for the classes that are instantiated through Guice and there are certain limitations.

## Example application

Lets finally try to use these libraries and see how it all works in a real application. The example project is hiding in the [GitHub Repo](https://github.com/htimur/play_metrics_example).

I'm using the `activator play-scala` template with sbt plugin. We should add `JCenter` resolver and dependencies, in the end it will look something like:

{% gist htimur/82e425df2a5df08dc91b22548c781dea %}

For the example I'll be using the `Console reporter`, so let's add the configuration to our `application.conf`.

{% gist htimur/ec8cc91c398f54c03882b5ac84fc6bab %}

As you can see I disabled `jvm` and `logback` metrics, and added the reporter that will periodically report metrics to `stdout` every `10 seconds`.

And we already can start using annotations. I'll annotate the `index` action of `HomeController`:

{% gist htimur/0ad1413299371fe99d5a253a9d013e18 %}

In real application you don't have to use all annotations at once, as `@Timed` will already create `Counter` and `Meter` metrics.

After we start the app and access the `Main Page`, it will output metrics to `stdout` as:

{% gist htimur/8267596c6d7e8cb75a414900f492420d %}

And of course you still can access the metrics through REST api, by adding route configuration to your `routes` file:

{% gist htimur/6bc260a8c7be9a908f5765acd41bc9f9 %}

## What is next?

##### Health Checks

Metrics also provides you a way to perform application health checks(a small self-tests) and this can be a great addition. For more information check the [docs](http://metrics.dropwizard.io/3.1.0/manual/healthchecks/).

##### More reporters

To build a proper environment there should be more ways to report the metrics. This can be a good way to go as well.

##### Better Future support

Currently if you want to measure the execution time of the `Future` it should be done manually. This can be a good improvement.

##### Hdrhistogram support

[Hdrhistogram](http://hdrhistogram.org/) provides alternative high quality reservoir implementations which can be used in histograms and timers.
