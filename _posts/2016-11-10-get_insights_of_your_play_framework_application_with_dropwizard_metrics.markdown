--- 
title: "Get insights of your Play Framework application with Dropwizard Metrics" 
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
 
At some point of the application development, all of us are coming to the point, when we are in need of more insights of what is happening inside of our applications or monitoring of the application. In case of Play Framework there is already drop in solution provided by great open source tool [Kamon](http://kamon.io/) and it's module [kamon-play](http://kamon.io/integrations/web-and-http-toolkits/play/). 
 
But today we are going to talk about alternative solution, [Drowizard Metrics](http://metrics.dropwizard.io) formerly known as `Codahale Metrics`, and it's integration and usage with Play Framework. 
 
## Integration 
 
Well, at this point I started to look around wondering if there are any libraries that could provide integration of these two. 
 
I found multiple libraries, but not complete though. For example: 
 
* [metrics-scala](https://github.com/erikvanoosten/metrics-scala) - Great library, neat API and Scala support, but in terms of Play Framework, just not enough. 
* [metrics-play](https://github.com/kenshoo/metrics-play) - One of the first modules that Google is trying to `satisfy you and get away`, but it's already abandoned and isn't compatible with latest versions of the Play Framework and Metrics. There is a [fork](https://github.com/breadfan/metrics-play) though, that is updated to the latest library versions, so I decided to give it a try. 
 
Unfortunately [metrics-play](https://github.com/breadfan/metrics-play) module have very basic support of the environment Dropwizard Metrics provides. It can be enough if you just need basic metrics which are exposed through REST api, but I had higher requirements and ended up extending the functionality of the module and building following modules: 
 
* [metrics-reporter-play](https://github.com/htimur/metrics-reporter-play) - Metrics reporter integration with Play Framework. 
* [metrics-annotation-play](https://github.com/htimur/metrics-annotation-play) - Metrics Annotations Support for Play Framework through Guice AOP. 
 
That's what we are going to talk next. 
 
#### Metrics reporter integration with Play Framework 
 
> Metrics provides a powerful toolkit of ways to measure the behavior of critical components in your production environment. 
 
As well as expose measured data through reporters, that's a great way of pushing the stats from your application to preferred storage backend. 
 
For the moment of writing the article, the supported reporters are: 
 
* console - Reports metrics periodically to the console. 
* graphite - Reports metrics periodically to Graphite. 
 
But Metrics library and community also provide various reporters like `Ganglia Reporter`, `CSV Reporter`, `InfluxDB Reporter`, `ElasticSearch Reporter` and other. 
 
Adding factories for reporters is no-brainer. 
 
#### Metrics Annotations Support for Play Framework through Guice AOP. 
 
By default to create and track metrics, we should invoke metric registry, create metrics and so on. Lets take a look: 
 
{% highlight scala %} 
def doSomethingImportant() = { 
    val timer = registry.timer(name(classOf[WebProxy], "get-requests")) 
    val context = timer.time() 
    try // critical business logic 
    finally context.stop() 
} 
{% endhighlight %} 
 
To keep it `DRY` there are annotations that you can use, the module will create and appropriately invoke a Timer for `@Timed`, a Meter for `@Metered`, a Counter for `@Counted`, and a Gauge for `@Gauge`. `@ExceptionMetered` is also supported, this creates a Meter that measures how often a method throws exceptions. 
 
The above example can be rewritten like: 
 
{% highlight scala %} 
@Timed 
def doSomethingImportant = { 
    // critical business logic 
} 
{% endhighlight %} 
 
or you can even annotate the class, to create metrics for all declared methods of it: 
 
{% highlight scala %} 
@Timed 
class SuperCriticalFunctionality { 
    def doSomethingImportant = { 
        // critical business logic 
    } 
} 
{% endhighlight %} 
 
Of course this is supported only for the classes that are instantiated through Guice and there are certain limitations. 
 
## Example application 
 
Lets finally try to use these libraries and see how it works in real application. 
 
I'm using the `activator play-scala` template with sbt plugin. We should add `JCenter` resolver and dependencies, in the end it will look something like: 
 
{% highlight scala %} 
 
resolvers += Resolver.jcenterRepo 
 
libraryDependencies ++= Seq( 
  jdbc, 
  cache, 
  ws, 
  "de.khamrakulov.metrics-reporter-play" %% "reporter-core" % "1.0.0", 
  "de.khamrakulov" %% "metrics-annotation-play" % "1.0.2", 
  "org.scalatestplus.play" %% "scalatestplus-play" % "1.5.1" % Test 
) 
{% endhighlight %} 
 
For the example I'll be using the `Console reporter`, so lets add the configuration to our `application.conf`. 
 
{% highlight json %} 
metrics { 
  jvm = false 
  logback = false 
 
  reporters = [ 
    { 
      type: "console" 
      frequency: "10 seconds" 
    } 
  ] 
} 
{% endhighlight %} 
 
As you can see I disabled `jvm` and `logback` metrics, and added the reporter, that will periodically report metrics to `stdout` every `10 seconds`. 
 
And we already can start using annotations. I'll annotate the `index` action of `HomeController`: 
 
{% highlight scala %} 
@Singleton 
class HomeController @Inject() extends Controller { 
 
  @Counted(monotonic = true) 
  @Timed 
  @Metered 
  def index = Action { 
    Ok(views.html.index("Your new application is ready.")) 
  } 
 
} 
{% endhighlight %} 
 
In real application you don't have to use all annotations at ones, as `@Timed` will already create `Counter` and `Meter` metrics.
 
After we start the app and access the `Main Page`, it will output metrics to `stdout` as: 
 
{% highlight %} 
-- Counters -------------------------------------------------------------------- 
controllers.HomeController.index.current 
             count = 1 
 
-- Meters ---------------------------------------------------------------------- 
controllers.HomeController.index.meter 
             count = 1 
         mean rate = 0.25 events/second 
     1-minute rate = 0.00 events/second 
     5-minute rate = 0.00 events/second 
    15-minute rate = 0.00 events/second 
 
-- Timers ---------------------------------------------------------------------- 
controllers.HomeController.index.timer 
             count = 1 
         mean rate = 0.25 calls/second 
     1-minute rate = 0.00 calls/second 
     5-minute rate = 0.00 calls/second 
    15-minute rate = 0.00 calls/second 
               min = 14.59 milliseconds 
               max = 14.59 milliseconds 
              mean = 14.59 milliseconds 
            stddev = 0.00 milliseconds 
            median = 14.59 milliseconds 
              75% <= 14.59 milliseconds 
95% <= 14.59 milliseconds 
98% <= 14.59 milliseconds 
99% <= 14.59 milliseconds 
99.9% <= 14.59 milliseconds 
{% endhighlight %} 
 
And of course you still can access the metrics through REST api, by adding route configuration to your `routes` file: 
 
{% highlight %} 
GET /admin/metrics com.kenshoo.play.metrics.MetricsController.metrics 
{% endhighlight %} 
 
## What is next 
 
##### Health Checks 
 
Metrics also provides your a way to perform application health checks(a small self-tests) and this can be a great addition. For more information check the [docs](http://metrics.dropwizard.io/3.1.0/manual/healthchecks/). 
 
##### More reporters 
 
To build a proper environment there should be more ways to report the metrics. This can be a good way to go as well. 
 
##### Better Future support 
 
Currently if you want to measure the execution time of the `Future` it should be done manually. This can be a good improvement. 
 
##### Hdrhistogram support 
 
[Hdrhistogram](http://hdrhistogram.org/) provides alternative high quality reservoir implementations which can be used in histograms and timers.