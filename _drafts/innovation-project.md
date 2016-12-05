---
layout: post
title: Building and shipping Raspberry-flavoured Microservices 
tags: [Microservices, Docker, RaspberryPi]
---

If you're like me, then you love reading and discussing papers, blogs and books on distributed systems.
But all theory is grey. It comes a time when you want to take things in your own hands and start putting the latest
and hottest tools and frameworks into action. That's exactly what has happened to me by the end of the last winter
term and finally resulted in a individual project I worked on over the past few months.

<!--more-->


## It all started with some RaspberryPis..

At our system engineering course in winter term 2015/16, I worked on a project along with two fellow students of mine. 
In short, our goal was building a Docker cluster consisting of several RaspberryPis and an Intel NUC from scratch. 
Although putting that into practice turned out to be more difficult than we thought at the beginning, we finally succeeded. 
In case you're interested in the details of what exactly we did and how it's been going, you can start 
[here](https://blog.mi.hdm-stuttgart.de/index.php/2016/03/10/more-is-always-better-building-a-cluster-with-pies/).<br/>
After we had finished, it came to my mind that simply leaving it to that without even having made use of our
new platform would be a pity. After having put so much effort in getting the Pi cluster up and running, I really wanted
to see this thing in action. 

### The idea
 
Roughly at the same time, I focused on microservices as well as the Netflix' open source stack in the context
of our *Ultra Large Scale Systems* course. Due to their special requirements, Netflix built and published many 
in-house tools which drive their scalable and highly available service infrastructure. So why not set up a tiny 
distributed applications in order to see these things in action? On top of that, why not integrating this into a 
small CI/CD cycle for testing purposes? To me, this seemed like a reasonable project that could directly be built on top
of our existing RaspberryPi cluster.

### The starting position

The cluster I started off with was made up of six [RaspberryPis](https://www.raspberrypi.org/) 
(version 2, Model B), running [Hypriot](http://blog.hypriot.com/) version 0.6 "Hector". As a cluster manager node, 
we additionally set up a [Intel NUC](http://www.intel.de/content/www/de/de/nuc/products-overview.html) with the
(at this time) latest Debian "Jessie" release. This mini computer was intended to act as the cluster workhorse, running 
all the central services which are essential for cluster management.
This basic architecture can be viewed in figure 1.<br/>
On top of our self-made "cloud platform", we ran a [Docker](https://www.docker.com/) swarm with every node having 
Docker Engine version 1.10 installed. Notice that our cluster does not use the new 
[Docker Swarm mode](https://docs.docker.com/engine/swarm/) which has been released with Docker 1.12, but instead 
relies on the legacy swarm implementation. I will come back to the differences and how this influenced my project
later on.

<div class="image-div">
    <img src="{{site.url}}/assets/2016-11-12/system_architecture.png" class="image-with-caption"/><br/>
    <small class="img-caption"><span>Figure 1: </span>Target platform consisting of a Intel
    NUC as the "cluster manager" as well as a Pi cluster of "workers".</small>
</div>

## Distributed systems patterns

In order to keep things manageable, I decided on three essential and popular distributed systems patterns I wanted 
to test in a practical manner by means of my demo application. These are the ones I focused on:

* __Service Discovery Pattern:__ This pattern plays an important role in distributed systems as it allows as to keep
our services loosely coupled from each other. What that means is that any service A, which depends on a second service
B, should not be statically bound to B's physical network location (i.e. IP address and port). Instead, A should be
able to "lookup" B by means of a logical name as soon as it wants to make a request. For that purpose, an additional
service is needed which keeps track of B's network address and makes it accessible via an unique name. A service 
that fulfills these requirements is what we call a "Service Discovery". 

* __Load Balancing:__ When load becomes heavy, a single instance of a service running on a dedicated host might not 
be capable of serving all the incoming requests. In order to prevent a server from having to handle too much work
and going down in the worst case, load should be split across a sufficient amount of replicas, which do exactly the
same thing. Load balancing can be implemented either on client or server side, and also on different layers of the 
network stack.

* __Circuit Breaker Pattern:__ In a distributed systems environment, every component can fail at any point in time. 
Coming back to our exemplary services A and B, imagine what would happen if B has crashed or continuously responds 
very slowly. When
relying on a blocking communication protocol (like HTTP) with single-thread-per-request servers, it would not take
much time until a large amount of A's threads keep waiting for B to respond. Since the server process inflates
because it is unable to free any resources, it will finally get killed by the operating system. As a consequence, 
we need to design distributed applications in a way that they can survive scenarios where other services are
unavailable. By integrating a "circuit breaker" into our applications, the system is able to deal with unexpected 
failures of remote services and becomes "resilient".

In the next section, we will take a short look at what the application I developed looks like. With that in mind,
we can move forward to the frameworks and tools which provide an implementation of these patterns and see how 
these can be leveraged to provide an app with real resilience.
  
  
## The distributed weather service

Since my focus was planned to be more on some of the properties of my distributed application than on the business 
logic, I decided on keeping things as simple as possible. The idea I came up with was implementing an application which
takes some user input (e.g. a zip code) and uses that to query a third-party weather API for the latest corresponding
weather data. Sounds simple enough, doesn't it? In order to end up with a distributed architecture, I planned to break 
that down into two components:

* __Weather Consumer Service:__ The consumer service constitutes the entry point into the application. It provides a
simple GUI which takes a zip code as well as an optional country code as user input and subsequently sends a HTTP
GET request to a producer service which responds with the desired data. After the response has arrived, the result
gets displayed on the GUI.
 
<div class="image-div"> 
    <img src="{{site.url}}/assets/2016-11-12/consumer_1.png" class="image-with-caption"/><br/>
    <small class="img-caption"><span>Figure 2: </span>Weather app entry point.</small>
</div>

* __Weather Producer Service:__ This second component of my application is the actual workhorse which receives
HTTP requests from the consumer service, contacts the [OpenWeatherMap](http://openweathermap.org/) service, tailors 
the response to a subset of information and finally sends it to the consumer service in JSON format.
 


## Microservices with Spring Boot

Just to tell you from the start: Since all the Netflix tools I applied to the weather application are written in Java,
I had to stick with Java (or at least any other JVM-related language, e.g. Scala) for development. One of the advantages
of using Java for my application was that I could use [Spring Boot](https://projects.spring.io/spring-boot/) in order
to implement standalone applications with minimal overhead. You will have understood what I mean by this after you've
taken a look at the following code snippet:

{% highlight java %}
@SpringBootApplication
public class SampleApp {
        
   @RequestMapping("/sayHello")
   @ResponseBody
   String hello() {
      "Hello World!";
   }
        
   public static void main(String[] args) {
      SpringApplication.run(SampleApp.class);
   }
  
}
{% endhighlight %}


Believe it or not, this is a fully functional web application. Spring Boot ships with an embedded Tomcat server and
lots of reasonable default configurations which make bootstrapping a simple application very easy.
Of course you previously need to setup the project with your favorite dependency management system (Maven or Gradle) 
before you can get started, but this is also far from being difficult. Once you're done, you can pack your entire
application into a single JAR file and execute it from the command line. If you're interested in the details of
Spring Boot, I recommend you to take a look at the docs.<br/>
Beyond that, the crucial factor for my decision to build on top of Spring Boot was that it already comes with
built-in support for the Netflix OSS tools I intended to apply to my project. 
[Spring Cloud Netflix](https://cloud.spring.io/spring-cloud-netflix/) offers these tools as individual 
module and significantly reduces the costs for integrating these into existing applications. I will go deeper into 
that during the next sections.  
 

## Hystrix - A circuit breaker implementation for microservices

Within our distributed weather service, there're actually two critical parts where bad things could happen:

1. The consumer service relies on the network to get in contact with the producer service. What might happen here
 is that no healthy producer instance is available or the entire network is down. Since I deployed both endpoints
 of the app within a single and rather reliable internal network, the latter might not be so probable. Nevertheless,
 the consumer must be prepared for getting no response from the producer side.
 
2. In the case of our producer service which communicates with the remote _OpenWeatherMap API_ over the internet, 
things 
are much more different. Because we don't have any influence on the availability of any third party service, we
always have to expect that servers are down just when we reach out to them. 

So what does that mean? Clearly, what we need is a way to make our services "fault tolerant", which is especially
true for our producer service which relies on a remote API on the internet. Above, I already introduced the 
Circuit Breaker Pattern and gave a short outline of what it is about. So let's take a closer at that.
 

### The Circuit Breaker Pattern 

Considering an electric circuit, a circuit breaker is a component which interrupts the circuit in case the amperage
exceeds a certain limit. In this way, it protects the electrical loads within the circuit from being damaged.  
You might ask yourself: What does this scenario have in common with microservices? The answer is: Quite a lot, 
as you will see now.

<div class="image-div"> 
    <img src="http://martinfowler.com/bliki/images/circuitBreaker/sketch.png" class="image-with-caption"/><br/>
    <small class="img-caption"><span>Figure 3: </span>How a circuit breaker works (http://martinfowler.com/bliki/images/circuitBreaker/sketch.png)</small>
</div>

By default, application servers like Tomcat spawn a new thread for every incoming request. In case a thread has to
wait for I/O (reading or writing to a database or file) or responses to network requests, it gets blocked by the 
operating system until the operation is finished. But what happens if the blocking operation takes long or doesn't
complete at all because the disk or the network is unavailable? Without further ado, actually nothing would happen,
leaving the waiting thread suspended forever.  
One approach to deal with this is forgetting about the waiting thread and simply repeat the request, hoping that
things go well this time. A new thread will be created and all operations will get started again. 
However, thinking of systems where thousands of requests per second rain down on a single service, this is not a great idea. 
You will finally end up with a huge mass of threads, all waiting for getting a job done which can't be concluded as 
long as any other required service is unreachable. The server process allocates more and more memory and finally 
crashes.  
To make matters worse, even a single slow or unavailable server impacts the whole system since in turn
other depending services will block, waiting for it to respond. In the end, your application will literally starve to 
death and finally come to a halt.  
The Circuit Breaker Pattern we already saw in terms of electrical circuits can help us here. Generally speaking, we can
supplement the weather producer service with a component which prevents our application threads from running into
useless requests assumed that we know that for some reason, the OpenWeatherMap API can't be reached at the moment. 
There're different strategies we can apply:

* __Fail fast:__ If a remote service is slow or completely unavailable, a client requesting it can be returned 
an error message after a predefined timeout (e.g. 100 ms) has elapsed. Further requests will not be executed at all but
immediately result in the same error message being returned by the circuit breaker.

* __Fail silent:__ As an alternative, one could also return an empty response instead of an error message in case a
remote service can't be reached. The advantage of this approach is that a client might not event notice that a single 
service has crashed, particularly if the final result is composed of the responses of several remote calls (fan out). 

* __Custom Fallback:__ Instead of confronting a client with an error message or an empty response, we can also try to 
mitigate the outage of any service by determining an alternative behavior. For example, we could implement a 
_fallback method_ which performs a cache lookup and provides the client with old data if possible. If that approach 
actually makes sense heavily depends on the use case.

In all these cases, the circuit breaker is in "open" state, where it blocks all outgoing requests to a certain server in
the network. In this way, it not only prevents depending services from starvation, but also gives the network or the 
remote server the change to possibly recover and return into operational state. Assuming that a crashed server reboots 
and comes to life again, the circuit breaker will gradually start to let requests pass instead of simply re-open
the gate. Thereby, the restored server can be protected from instantly having to deal with a large amount of tasks, 
which might badly increase the risk of another crash.  




### Bringing Hystrix to our weather service

Let's take a look at how Hystrix is employed by the sample application using the example of our weather producer app. 
Since I relied on _Spring Cloud Netflix_ to integrate the Netflix OSS stack into the weather app, enabling Hystrix 
was not difficult. All that's required is the following two steps:

1. Add the _@EnableCircuitBreaker_ annotation to the Spring Boot main class.
2. For making a method being backed by Hystrix, supply it with the _@HystrixCommand_ annotation. 

As indicated by the _@HystrixCommand_ annotation, Hystrix is built upon the 
[Command design pattern](https://en.wikipedia.org/wiki/Command_pattern). Every method which has this annotation is
wrapped by a command object and executed within a new thread while the original thread is blocked. In case a predefined
timeout has elapsed (e.g. due to a unavailable remote service) all that Hystrix does is give back control to the 
original thread. From that moment on, we're free decide how to continue. For my weather service, I determined to 
implement a fallback which performs a cache lookup and either returns the latest result for a certain zip code if 
available, or returns an empty response object. For the sake of simplicity, I used a _java.util.HashMap_ per service 
instance 
as a volatile in-memory cache. In advanced scenarios, establishing a smarter solution based on [Redis](https://redis.io/)
or [Memcached](https://memcached.org/) that can be shared across multiple service instances and is not bound to the 
lifecycle of the applications makes much more sense.  
This is the method from the producer service which performs an HTTP request to the OpenWeatherMap API:


{% highlight java %}
@HystrixCommand(fallbackMethod = "weatherApiFallback")    
public WeatherResponseObject 
    getWeatherDataByZipCodeAndCountry(String zipCode, 
                                      String countryCode) 
 {
    URI openWeatherMapURI = 
        buildUriWithRequestParams(zipCode, countryCode);
    WebResource webResource = 
        prepareWebResource(openWeatherMapURI);
    ClientResponse clientResponse = 
        dispatchRequest(webResource);
        
    if (isResponseOk(clientResponse)) {
        WeatherResponseObject result = 
            responseFormatter.format(clientResponse);
        weatherDataCache.store(zipCode, result);
        return result;
    } else {
        throw new BadApiResponseException(
            String.format("Bad response: %s (%d)",
                clientResponse.getStatusInfo()
                .getReasonPhrase(), 
                clientResponse.getStatus()));
    }
 }  
{% endhighlight %}
  
  
The _weatherApiFallback_ method, which is specified by means of an annotation parameter and must have the exact same 
signature as the Hystrix-wrapped method, looks as follows:
  
{% highlight java %}
    public WeatherResponseObject 
        weatherApiFallback(String zipCode, String countryCode) {
        
        WeatherResponseObject responseObject = 
            weatherDataCache
            .lookup(zipCode)
            .orElse(WeatherResponseObject.getEmpty());
            
        if (responseObject.isCached()) {
            responseObject.setMessage(
                "CAUTION - Response out of date!");
        }
        
        return responseObject;
    }
{% endhighlight %}

If there's, for whatever reason, just a cached result available, the fallback method attaches a message to the result
object telling the client that this result is actually out of date. With that in mind, he or she is free to repeat 
the request when the OpenWeatherMap API (hopefully) responds sooner or later.  
The same approach based on Hystrix is used in order to protect the weather consumer from a failing producer service. 
    
 
    
    
## Loosely coupling weather consumer and producer
     
I shortly talked a little bit about the _Service Discovery Pattern_ above. In the case of our weather application, 
applying this pattern means to keep consumer and producer highly decoupled. What this means is that we don't want our
consumer to be statically bound to IP and port of a producer service instance. Using hard-coded URLs in such a 
scenario is a very bad idea: If the producer fails for some reason, the consumer code would have to get changed in 
order to point to another healthy instance. Oh dear, that sounds horrible ...  


### Service Discovery to the rescue

By establishing a service discovery, that issue can be circumvented. In a nutshell, it works this way:
    
+ Services pass their network address to a service discovery and register them under a certain name. There can be
     more than one instance of a single service.
     
+ Clients which want to reach out to a certain service can "lookup" its address by name at the service discovery. 
Thereby, they no longer have to be aware of the current network address which can even be changed without breaking 
clients.

So what does Netflix OSS offer us here? Netflix released a service discovery solution named 
[Eureka](https://github.com/Netflix/eureka), which they use to coordinate their services deployed on the Amazon AWS 
cloud. Eureka also ships with _Spring Cloud Netflix_ and can be used in the context of Spring Boot without much effort.
 
 
### Configuring and launching Eureka server  
 
An Eureka server can be started as a standalone Spring Boot application. All that we need is a main class which is 
annotated with _@EnableEurekaServer_:
 
{% highlight java %}
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApp {
              
   public static void main(String[] args) {
      SpringApplication.run(EurekaServerApp.class);
   }
}
{% endhighlight %} 

That's almost everything that has to be done. What's left is a little bit of configuration which has to be placed 
within the _application.yml_ file:

{% highlight yaml %}
server:
  port: 8761

eureka:
  instance:
    hostname: eureka
  client:
    registerWithEureka: false
    fetchRegistry: false
    serviceUrl:
      defaultZone: http://${eureka.instance.hostname}:
                                       ${server.port}/eureka/
{% endhighlight %}
 

There's a couple of interesting this here. The first issue I ran into was that I didn't turn off the Eureka server's 
client behavior. It's important to know that by default every Eureka server is also a Eureka client and searches for 
other peers after startup. However, once again I decided on keeping thing simple and restrained the service discovery
part to a single Eureka node running in standalone mode. Therefore, I fully disabled the client behavior by setting 
the appropriate attributes to "false" in the configuration file which keeps Eureka from throwing exceptions on 
execution.  


### Registering the producer service with Eureka 

Enhancing the producer service with Eureka client capabilities is also very easy and quite similar to the 
configuration we did for the server. As far as the app's main class is concerned, the only change we have to apply is
using the _@EnableEurekaClient_ annotation instead of the corresponding server annotation.    
The _application.yml_ also reflects some slight differences compared to the server config:

{% highlight yaml %}
spring:
  application:
    name: producer

server:
  port: 0

eureka:
  client:
    serviceUrl:
      defaultZone: http://eureka:8761/eureka/
  instance:
   preferIpAddress: true
{% endhighlight %}


The first noticeable change is that we defined the server port of the producer app to be "0". Actually, what this 
does is making Spring Boot select a random free port by itself. This is awesome in many ways because on the one hand,
that spares us the effort to check a host for a free port manually if we'd like to start several instances of the 
producer service on a single host. Moreover, we actually don't even care about 
port the producer application runs on because we don't have to know about that. Since every producer registers with our 
service discovery, we can do a lookup and fetch the network address if necessary.   
You might have noticed that I address the host "eureka" within the _serviceUrl_ attribute value. Once again, this 
keeps me from having to hard code an IP address, in this case the Eureka server host's one. What allows me to do that
 is a networking feature provided by Docker. I will cover this later.
   

### Looking up the producer service from the consumer-side   

{% highlight java %}
    @HystrixCommand(fallbackMethod = "producerRequestFallback")
       public WeatherResponseObject requestProducerService(String zipCode, String countryCode) {
           ServiceInstance producerInstance = loadBalancerClient.choose(PRODUCER_SERVICE_ID);
   
           WeatherResponseObject response = new RestTemplate().getForObject(buildURI(producerInstance),
                   WeatherResponseObject.class, toRequestParamMap(zipCode, countryCode));
   
           weatherDataCache.store(zipCode, response);
   
           return response;
       } 
{% endhighlight %}



## CI workflow



## Sources

* Netflix, Inc. (2016). _Netflix/eureka_. [online] Available at:
[https://github.com/Netflix/eureka](https://github.com/Netflix/eureka)
[Accessed 05 Dec. 2016].

* Netflix, Inc. (2016). _Netflix/Hystrix_. [online] Available at:
[https://github.com/Netflix/Hystrix](https://github.com/Netflix/Hystrix)
[Accessed 05 Dec. 2016].

* Netflix, Inc. (2016). _How to Use_. [online] Available at:
[https://github.com/Netflix/Hystrix/wiki/How-To-Use](https://github.com/Netflix/Hystrix/wiki/How-To-Use)
[Accessed 04 Dec. 2016].

* Pivotal Software. (2016).  _Spring Cloud Netflix_. [online] Available at: 
[http://cloud.spring.io/spring-cloud-netflix/spring-cloud-netflix.html](http://cloud.spring
.io/spring-cloud-netflix/spring-cloud-netflix.html)  
[Accessed 05 Dec. 2016].