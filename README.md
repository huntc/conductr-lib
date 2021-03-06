# Typesafe ConductR Bundle Library

[![Build Status](https://api.travis-ci.org/typesafehub/conductr-lib.png?branch=master)](https://travis-ci.org/typesafehub/conductr-lib)

## Introduction

This project provides a number of libraries to facilitate [ConductR](http://typesafe.com/products/conductr)'s status service and its service lookup service. Note that usage of the libraries in your code is entirely benign when used outside of the context of ConductR i.e. you will find that your applications and services will continue to function normally when used without ConductR. We have also designed the libraries to be a convenience to ConductR's REST and environment variable based APIs, and to have a very low impact on your code.

Add one of the following libraries to your project.

* `"com.typesafe.conductr" %  "conductr-bundle-lib"         % "1.4.3"`
* `"com.typesafe.conductr" %  "java-conductr-bundle-lib"    % "1.4.3"`
* `"com.typesafe.conductr" %% "scala-conductr-bundle-lib"   % "1.4.3"`
* `"com.typesafe.conductr" %% "akka23-conductr-bundle-lib"  % "1.4.3"`
* `"com.typesafe.conductr" %% "akka24-conductr-bundle-lib"  % "1.4.3"`
* `"com.typesafe.conductr" %% "play23-conductr-bundle-lib"  % "1.4.3"`
* `"com.typesafe.conductr" %% "play24-conductr-bundle-lib"  % "1.4.3"`
* `"com.typesafe.conductr" %% "play25-conductr-bundle-lib"  % "1.4.3"`
* `"com.typesafe.conductr" %% "lagom10-conductr-bundle-lib" % "1.4.3"`

You will also require a resolver:

```scala
resolvers += Resolver.bintrayRepo("typesafe", "maven-releases")
```

Also note that the examples here use the following import to conveniently build the JDK `URI` and `URL` types. 

```scala
import com.typesafe.conductr.bundlelib.scala.{URL, URI}
```

## Table of contents

* [conductr-bundle-lib](#conductr-bundle-lib)
* [scala-conductr-bundle-lib](#scala-conductr-bundle-lib)
* [akka[23|24]-conductr-bundle-lib](#akka2324-conductr-bundle-lib)
* [play[23|24]-conductr-bundle-lib](#play2324-conductr-bundle-lib)
* [play25-conductr-bundle-lib](#play25-conductr-bundle-lib)
* [lagom10-conductr-bundle-lib](#lagom10-conductr-bundle-lib)

## conductr-bundle-lib

This library provides a base level of functionality mainly formed around constructing the requisite payloads for ConductR's RESTful services. The library is pure Java and has no dependencies other than the JDK.

Two services are covered by this library:

* `com.typesafe.conductr.bundlelib.LocationService`
* `com.typesafe.conductr.bundlelib.StatusService`

### Location Service

ConductR's location service is able to respond with a URI declaring where a given service (as named by a bundle component's endpoint) resides. The http payload can be constructed as follows:

```java
HttpPayload payload = LocationService.createLookupPayload("someservice")
```

The `HttpPayload` object may then be queried for elements that will help you make an http request. These methods are:

* `getUrl`
* `getRequestMethod`
* `getFollowRedirects`

When processing the http response you should check for the following http status codes:

* 307 - temporary redirect - the service can be found at the location indicated by the `Location` header. `Cache-Control` may also be supplied to indicate how long the location may be cached for.
* 404 - not found - the service cannot be located at this time

You should also prepare for timing out on a request and process as per a 404.

### Status Service

Conduct's status service is required by a bundle component in order to signal when it has started. A successful startup is anything that the application is required to do to become available for processing. For example, this may involved validating configuration. The http payload can be constructed as follows:

 ```java
 HttpPayload payload = StatusService.createSignalStartedPayload()
 ```

The `HttpPayload` object may then be queried for elements in the same way as for the location service of the previous section.

When processing the http response you should check for the following http status codes:

* 2xx - success - any 200 series response is a success meaning that ConductR has successfully acknowledged the startup signal
* xxx - failure - anything else constitutes an error and you should cause the bundle component to exit

You should also prepare for timing out on a request and exit the bundle component if this occurs.

## scala-conductr-bundle-lib

This library provides a reactive API using only Scala and Java types. There are no dependencies other than `conductr-bundle-lib`, Scala and the JDK and it is designed to be used where there is no Akka or Play dependency in your application.

As with `conductr-bundle-lib` there are two services:

* `com.typesafe.conductr.bundlelib.scala.LocationService`
* `com.typesafe.conductr.bundlelib.scala.StatusService`

Please read the section on `conductr-bundle-lib` for an introduction to these services.

### LocationService

The LocationService looks up service names and processes HTTP's `307` "temporary redirect" responses to return the location of the resolved service (or a `404` if one cannot be found). Many HTTP clients allow the following of redirects, particularly when either of the `HEAD` or `GET` methods are used (other methods may be considered insecure by default). Therefore if the service you are locating is an HTTP one then using a regular HTTP client should require no further work. Here is an example of using the [Dispatch](http://dispatch.databinder.net/Dispatch.html) library:

```scala
val svc = LocationService.getLookupUrl("someservice", URL("http://127.0.0.1:9000/someservice"))
val svcResp = Http.configure(_.setFollowRedirects(true))(url(svc.toString).OK)
```

The above declares an `svc` val which will either be the one that ConductR provides, or one to use for development running on your machine.

When using HTTP clients, consider having the client cache responses. ConductR will return Cache-Control header information informing the client how to cache.

#### Non HTTP service lookups

If the service you require is not HTTP based then you may use the `LocationService.lookup` function. The following code illustrates how a service may be located in place of creating and dispatching your own payload. The sample also shows how to use a cache provided specifically for these lookups (note use com.typesafe.lib.scala for 1.2 of this library onwards):

```scala
// This will require an implicit ConnectionContext to
// hold a Scala ExecutionContext. There are different
// ConnectionContexts depending on which flavor of the
// library is being used. For the Scala flavor, a Scala
// ExecutionContext is composed. The ExecutionContext
// is needed as "service" is returned as a Future.
// For convenience, we provide a global ConnectionContext
// that may be imported.
import com.typesafe.conductr.bundlelib.scala.ConnectionContext.Implicits.global

val locationCache = LocationCache()

val service = LocationService.lookup("someservice", URI("tcp://localhost:1234"), locationCache)
```

`service` is typed `Future[Option[URI]]` meaning that an optional URI response will be returned at some time in the future. Supposing that this lookup is made during the initialisation of your program, the service you're looking for may not exist. However calling the same function later on may yield the service. This is because services can come and go. Note that the fallback URI of `"tcp://localhost:1234"` will be returned if this function is called upon when started outside of ConductR.

The service response constitutes a URI that describes its location.

#### Static service lookup

Some bundle components cannot proceed with their initialisation unless the service can be located. We encourage you to re-factor these components so that they look up services at the time when they are required, given that services can come and go. However if you are somehow stuck with this style of code then you may consider the following blocking code as a temporary measure:

```scala
val resultUri = Await.result(
  LocationService.lookup("someservice", URI("http://127.0.0.1:9000"), locationCache),
  sometimeout)
val serviceUri = resultUri.getOrElse(System.exit(70))
```

In the above, the program will exit if a service cannot be located at the time the program initializes; unless the program has not been started by ConductR in which case an alternate URI is provided.

### StatusService

The following code illustrates how your bundle component should register its initial health with ConductR. Calling this function is to be done in place of creating and dispatching your own payload:

```scala
StatusService.signalStartedOrExit()
```

In general, the return value of `signalStartedOrExit` is not used and your program proceeds. If ConductR fails to reply, or replies with an error status then this bundle component will exit.

In case you are interested, the function returns a `Future[Option[Unit]]` where a future `Some(())` indicates that ConductR has successfully acknowledged the startup signal. A future of `None` indicates that the bundle has not been started by ConductR.

## akka[23|24]-conductr-bundle-lib

This library provides a reactive API using [Akka Http](http://akka.io/docs/) and should be used when you are using Akka. The library depends on `scala-conductr-bundle-lib` and can be used for both Java and Scala.

As with `conductr-bundle-lib` there are these two services:

* `com.typesafe.conductr.bundlelib.akka.LocationService`
* `com.typesafe.conductr.bundlelib.akka.StatusService`

and there is also another:

* `com.typesafe.conductr.bundlelib.akka.Env`

Please read the section on `conductr-bundle-lib` and then `scala-conductr-bundle-lib` for an introduction to these services. The `Env` one is discussed in the "Akka Clustering" section below.

Other than the `import`s for the types, the only difference in terms of API are usage is how a `ConnectionContext` is established. A `ConnectionContext` for Akka requires an implicit `ActorSystem` or `ActorContext` at a minimum e.g.:

```scala
 implicit val cc = ConnectionContext()
```

There is also a lower level method where the `HttpExt` and `ActorMaterializer` are passed in:

```scala
implicit val cc = ConnectionContext(httpExt, actorMaterializer)
```

When in the context of an actor, a convenient `ImplicitConnectionContext` trait may be mixed in to establish the `ConnectionContext`. The next section illustrates this in its sample `MyService` actor.

### Static Service Lookup

As a reminder, some bundle components cannot proceed with their initialisation unless the service can be located. We encourage you to re-factor these components so that they look up services at the time when they are required, given that services can come and go. That said, here is a non-blocking improvement on the example provided for the `scala-conductr-bundle-lib`:


```scala
class MyService(cache: CacheLike) extends Actor with ImplicitConnectionContext {

  import context.dispatcher

  override def preStart(): Unit =
    LocationService.lookup("someservice", URI("http://127.0.0.1:9000"), cache).pipeTo(self)

  override def receive: Receive =
    initial

  private def initial: Receive = {
    case Some(someService: URI) =>
      // We now have the service

      context.become(service(someService))

    case None =>
      self ! PoisonPill
  }

  private def service(someService: URI): Receive = {
    // Regular actor receive handling goes here given that we have a service URI now.
    ...
  }
}
```

This type of actor is used to handle service processing and should only receive service oriented messages once its dependent service URI is known. This is an improvement on the blocking example provided before, as it will not block. However it still has the requirement that `someservice` must be running at the point of initialization, and that it continues to run. Neither of these requirements may always be satisfied with a distributed system.

### Java

The following example illustrates how status is signalled using the Akka Java API:

```java
ConnectionContext cc = ConnectionContext.create(system);
StatusService.getInstance().signalStartedOrExitWithContext(cc);
```

Similarly here is a service lookup:

```java
ConnectionContext cc = ConnectionContext.create(system);
LocationService.getInstance().lookupWithContext("whatever", URI("tcp://localhost:1234"), cache, cc)
```

### Akka Clustering

[Akka cluster](http://doc.akka.io/docs/akka/snapshot/scala/cluster-usage.html) based applications or services have a requirement where the first node in a cluster must form the cluster, and the subsequent nodes join with any of the ones that come before them (seed nodes). Where bundles share the same `system` property in their `bundle.conf`, and have an intersection of endpoint names, then ConductR will ensure that only one bundle is started at a time. Thus the first bundle can determine whether it is the first bundle, and subsequent bundles can determine the IP and port numbers of the bundles that have started before them.

In order for an application or service to take advantage of this guarantee provided by ConductR, the following call is required to obtain configuration that will be used when establishing your actor system:

```scala
import com.typesafe.conductr.bundlelib.akka.Env

val config = Env.asConfig
val systemName = sys.env.getOrElse("BUNDLE_SYSTEM", "MyApp1")
val app1 = ActorSystem(systemName, config.withFallback(ConfigFactory.load()))
```

Clusters will then be formed correctly. The above call looks for an endpoint named `akka-remote` by default. Therefore if you must declare the Akka remoting port as seed. The following endpoint declaration within a `build.sbt` shows how:

```scala
BundleKeys.endpoints := Map("akka-remote" -> Endpoint("tcp"))
```

In the above, no declaration of `services` is required as akka remoting is an internal, cluster-wide TCP service.

## play25-conductr-bundle-lib

> If you are using Play 2.5 then this section is for you. Otherwise jump below to the "play[23|24]-conductr-bundle-lib" section.

[sbt-conductr](https://github.com/typesafehub/sbt-conductr) is automatically adding this library to your Play project.

This library provides a reactive API using [Play WS](https://www.playframework.com/documentation/2.5.x/ScalaWS) and should be used when you are using Play. The library depends on `akka24-conductr-bundle-lib` and can be used for both Java and Scala. As per Play's conventions, `play.api` is used for the Scala API and just `play` is used for Java.

As with `conductr-bundle-lib` there are two services:

* `com.typesafe.conductr.bundlelib.play.LocationService` (Java) or `com.typesafe.conductr.bundlelib.play.api.LocationService` (Scala)
* `com.typesafe.conductr.bundlelib.play.StatusService` (Java) or `com.typesafe.conductr.bundlelib.play.api.StatusService` (Scala)

and there is also another:

* `com.typesafe.conductr.bundlelib.play.Env` (Java) or `com.typesafe.conductr.bundlelib.play.api.Env` (Scala)

Please read the section on `conductr-bundle-lib` and then `scala-conductr-bundle-lib` for an introduction to these services. The `Env` one is discussed in the section below. The major difference between the APIs for Play 2.5 and the other variants is that components are expected to be injected. For example, to use the `LocationService` in your controller (Scala):

```scala
class MyGreatController @Inject() (locationService: LocationService, locationCache: CacheLike) extends Controller {
  ...
  locationService.lookup("known", URI(""), locationCache)
  ...
}
```

The following components are available for injection:

* CacheLike
* ConnectionContext
* LocationService
* StatusService

Note that if you are using your own application loader then you should ensure that the Akka and Play ConductR-related properties are loaded. Here's a complete implementation (for Scala):

```scala
class MyCustomApplicationLoader extends ApplicationLoader {
  def load(context: ApplicationLoader.Context): Application = {
    val conductRConfig = Configuration(AkkaEnv.asConfig) ++ Configuration(PlayEnv.asConfig)
    val newConfig = context.initialConfiguration ++ conductRConfig
    val newContext = context.copy(initialConfiguration = newConfig)
    val prodEnv = Environment.simple(mode = Mode.Prod)
    (new GuiceApplicationLoader(new GuiceApplicationBuilder(environment = prodEnv))).load(newContext)
  }
}
```

## play[23|24]-conductr-bundle-lib

Please select the Play 2.3 or 2.4 variant depending on whether you are using Play 2.3 or Play 2.4 respectively.

[sbt-conductr](https://github.com/typesafehub/sbt-conductr) is automatically adding this library to your Play project.

This library provides a reactive API using [Play WS](https://www.playframework.com/documentation/2.3.x/ScalaWS) and should be used when you are using Play. The library depends on `akka23-conductr-bundle-lib` and can be used for both Java and Scala.

As with `conductr-bundle-lib` there are two services:

* `com.typesafe.conductr.bundlelib.play.LocationService`
* `com.typesafe.conductr.bundlelib.play.StatusService`

and there is also another:

* `com.typesafe.conductr.bundlelib.play.Env`

Please read the section on `conductr-bundle-lib` and then `scala-conductr-bundle-lib` for an introduction to these services. The `Env` one is discussed in the section below. Other than the `import`s for the types, the only difference in terms of API are usage is how a `ConnectionContext` is established. A `ConnectionContext` for Play requires an `ExecutionContext` at a minimum. For convenience, we provide a default ConnectionContext using the default execution context. This may be imported e.g. (note use com.typesafe.lib.play for 1.2 of this library onwards):

```scala
  import com.typesafe.conductr.bundlelib.play.ConnectionContext.Implicits.defaultContext
```

There is also a lower level method where the `ExecutionContext` is passed in:

```scala
implicit val cc = ConnectionContext(executionContext)
```

### Java

The following example illustrates how status is signalled using the Play Java API:

```java
ConnectionContext cc =
    ConnectionContext.create(HttpExecution.defaultContext());

  ...

StatusService.getInstance().signalStartedOrExitWithContext(cc);
```

Similarly here is a service lookup:

```java
ConnectionContext cc =
    ConnectionContext.create(HttpExecution.defaultContext());

  ...

LocationService.getInstance().lookupWithContext("whatever", new URI("tcp://localhost:1234"), cache, cc)
```

In order for an application or service to take advantage of setting important Play related properties, the following is required in order to associate ConductR configuration with that of Play and Akka:


#### Play 2.4

Note that if you are using your own application loader then you should ensure that the Akka and Play ConductR-related properties are loaded. Here's a complete implementation:

```scala
class MyCustomApplicationLoader extends ApplicationLoader {
  def load(context: ApplicationLoader.Context): Application = {
    val conductRConfig = Configuration(AkkaEnv.asConfig) ++ Configuration(PlayEnv.asConfig)
    val newConfig = context.initialConfiguration ++ conductRConfig
    val newContext = context.copy(initialConfiguration = newConfig)
    val prodEnv = Environment.simple(mode = Mode.Prod)
    (new GuiceApplicationLoader(new GuiceApplicationBuilder(environment = prodEnv))).load(newContext)
  }
}
```

#### Play 2.3

```scala
import play.api._
import com.typesafe.conductr.bundlelib.play.Env

object Global extends GlobalSettings {
  val totalConfiguration = super.configuration ++ Configuration(Env.asConfig)

  override def configuration: Configuration =
    totalConfiguration
}
```

## lagom10-conductr-bundle-lib

> If you are using Lagom 1.0.x then this section is for you.

[sbt-conductr](https://github.com/typesafehub/sbt-conductr) is automatically adding this library to your Lagom project. You don't need set any additional setting for your Lagom services.

Note that if you are using your own application loader then you should ensure that the Akka, Play and Lagom ConductR-related properties are loaded. Here's a complete implementation (in Scala):

```java
class MyCustomApplicationLoader extends ApplicationLoader {
  def load(context: ApplicationLoader.Context): Application = {
    val conductRConfig = Configuration(AkkaEnv.asConfig) ++ Configuration(PlayEnv.asConfig) ++ Configuration(LagomEnv.asConfig)
    val newConfig = context.initialConfiguration ++ conductRConfig
    val newContext = context.copy(initialConfiguration = newConfig)
    val prodEnv = Environment.simple(mode = Mode.Prod)
    (new GuiceApplicationLoader(new GuiceApplicationBuilder(environment = prodEnv))).load(newContext)
  }
}
```
