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

<br/>

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


<br/>

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
  
<br/>
  
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
 

<br/>

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
 
 
<br/> 

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
    weatherApiFallback(String zipCode, String countryCode) 
 { 
    WeatherResponseObject responseObject = 
        weatherDataCache
            .lookup(zipCode)
            .orElse(WeatherResponseObject.getEmpty());
            
    if (responseObject.isCached()) {
        responseObject
            .setMessage("CAUTION - Response out of date!");
        }
        
    return responseObject;
 }
{% endhighlight %}

If there's, for whatever reason, just a cached result available, the fallback method attaches a message to the result
object telling the client that this result is actually out of date. With that in mind, he or she is free to repeat 
the request when the OpenWeatherMap API (hopefully) responds sooner or later.  
The same approach based on Hystrix is used in order to protect the weather consumer from a failing producer service. 
    
 
<br/>    
    
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
other Eureka peers after startup (this mode is calles "peer mode"). However, once again I decided on keeping thing 
simple and restrained the service 
discovery part to a single Eureka node running in standalone mode. Therefore, I fully disabled the client behavior by
setting the appropriate attributes to "false" in the configuration file which starts Eureka in "standalone mode" and 
keeps it from from throwing exceptions on execution.  


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
keeps me from having to hard code an IP address, in this case the Eureka server host's one. What allows me to address
the Eureka server by name is a special networking feature provided by Docker. I will cover this later.
   


### Looking up the producer service from the consumer-side   

Now that we're able to start an arbitrary number of producer instances and register them with Eureka service discovery,
the consumer service must be enabled to perform _lookups_ thereby receive the network address of any available
producer service. For that purpose, Spring Cloud offers a client API for hitting Eureka and fetching discovery data. 
The special thing about that API is that there's no constraint to use it along with Eureka. If you prefer working 
with alternative service discovery tools like [Apache ZooKeeper](https://zookeeper.apache.org/) or [Consul by 
HashiCorp](https://www.consul.io/), you're free to do so. For my project, two reasons made me use Eureka: First, that
allowed me to stay within the Netflix OSS ecosystem. Second, Eureka offers a very nice GUI which is more practical 
for demo purposes than e.g. the ZooKeeper CLI (which I still consider very handy).   
    
    

{% highlight java %}
@HystrixCommand(fallbackMethod = "producerRequestFallback")
public WeatherResponseObject 
    requestProducerService(String zipCode, String countryCode) 
 {
    ServiceInstance producerInstance = 
        loadBalancerClient.choose(PRODUCER_SERVICE_ID);
   
    WeatherResponseObject response = new RestTemplate()
        .getForObject(
            buildURI(producerInstance),
            WeatherResponseObject.class, 
            toRequestParamMap(zipCode, countryCode)
        );
   
    weatherDataCache.store(zipCode, response);
   
    return response;
 } 
{% endhighlight %}



### Client-side load balancing with Ribbon

For my weather consumer service, I didn't apply the standard _DiscoveryClient_ implementation as described above. 
Instead, I chose to give _Ribbon_ a try, which is a Netflix library for client-side load balancing. Note that this 
does not mean that there's no load balancing offered by the _DiscoveryClient_ class. However, it's built-in load 
balancing capabilities are restricted to a naive round-robin approach, whereas Ribbon supports different load 
balancing rules, different protocols beyond HTTP ([gRPC](http://www.grpc.io/) support is work in progress) and also 
caching. 
Using Ribbon is very simple: For my Spring Boot consumer, I simply put the required dependencies on classpath and 
added the _@EnableEurekaClient_ annotation to the main class. Then, you can obtain a _LoadBalancerClient_ instance 
from the Spring context via dependency injection. That's it. From then on, we can ask the _LoadBalancerClient_ instance 
for a remote service address by passing it the name of the service we want to hit. Ribbon fetches the list of 
available servers from Eureka, performs its load balancing magic and returns a single _ServiceInstance_ object, which
contains IP address and port which can in turn be used to setup an HTTP request.
Since reaching out to RESTful APIs via HTTP often results in boilerplate code, Spring Cloud also integrates 
[Feign](https://github.com/OpenFeign/feign), a library which aims at keeping developers from writing the same code 
for speaking with remote APIs over and over again. Its approach is completely declarative, meaning that all that's 
left to the user is creating an Interface which provides the signature for the remote call. Using that as a basis, 
Feign cares about generating the application code on the fly and passing to the client. 
Although I think that using Feign can help building clean and slight applications, I resigned to integrate it into my
project. The reason for that simply is that in my opinion implementing remote calls in a verbose way instead of 
abstracting things away behind Feign provides a much better demo of what's really going on and helps to understand the 
steps which are performed (reaching out to service discovery, doing client-side load balancing, sending HTTP request). 
In a productive environment, using Feign would of course be a reasonable choice.        
    



<br/>

## Shipping the weather app to our Pi cluster 

At this point, I guess you have a rough impression of the tools I used as well as the decisions I made for my 
distributed demo application. The next step was developing a strategy for bringing the single services to the Docker 
cluster which, as I already mentioned at the beginning, already existed and was the motivation for doing this project
anyway. 


### Change happens

So let's briefly recap where I've been before I started my project:
     
 + The existing Docker cluster consisted of six RaspberryPis as well as a single Intel NUC.
 + The OS driving the Pis was [Hypriot](http://blog.hypriot.com/) Version 0.5 "Will" (I really like these guys for 
 their references on _Pirates of the Caribbean_).
 + On the NUC, we setup Debian "Jessie".
 + On the Pis as well as the NUC, we started with Docker Engine version 1.8.2, Docker Machine 0.4.1 and Docker Swarm 
 0.4.0.
 + HypriotOS Linux kernel was 4.1.8.
 + We established a Docker swarm, with the NUC as the master node and the RaspberryPis as a cluster of worker nodes. 
 + On top of that, the NUC as the swarm master node also ran [Shipyard](https://shipyard-project.com/), which 
 provides a very nice GUI for Docker container and swarm management. 
 
 
When we just finished that project in January 2016, this setup worked perfectly. However, problems emerged when I 
updated the whole cluster about six months later:
   
 + After having installed updates on every cluster node, the Intel NUC suddenly was on Docker 1.12. At the same time,
  I couldn't get beyond Docker 1.10 with the Hypriot version running on the Pi cluster.
 + As a consequence, building on top the new swarm mode which came with Docker Engine 1.12 was not possible in my case. 
 + I also realized there wasn't full backward compatibility between the Docker Machine versions on the NUC and the 
 Pis after the update.
 + In the meantime, Docker launched a new version of their image registry. This was not so good for me since the 
 Shipyard release running on the NUC did not support the new API. 
 
 
Just to make things clear: My original schedule for the project didn't respect the costs of setting up the entire 
cluster from scratch, including flashing every single Pi with the latest Hypriot release. Although that would have been 
the best approach in my opinion in order to start with a proper setup, I decided to live with that and make the best 
out of it. Unfortunately, I had almost finished the project when I learned that replacing the existing Docker 
installation on the Pis with the Raspbian release could have mitigated my problems 
(at this point, thanks to Hypriot team member Dieter Reuter).  
 
     

### Wrap the weather app in Docker images      

Luckily for me, taking the weather consumer service, the producer service as well as the Eureka service and building 
Docker images around them was very easy. I added Dockerfiles to the service's top level directories in order to be 
able to build images from the command line. The most interesting part here was finding proper base images for 
executing Java applications on ARM devices. At this point, the Hypriot team once more did great work and already 
published the _hypriot/rpi-java_ image on [Docker Hub](https://hub.docker.com/r/hypriot/rpi-java/). The Dockerfile I 
ended up with for the producer service looks like this:
      
{% highlight java %}
FROM hypriot/rpi-java
ADD ./target/producer.jar producer.jar
COPY api_key /tmp/api_key
ENTRYPOINT ["java", "-jar", "producer.jar"]
{% endhighlight %}       


You see that this Dockerfile is actually very simple. All it does is taking the JAR from the target directory (after 
the build), copy it into the image along with the OpenWeatherMap API key and finally start it up as soon as a 
container is created from that image. I won't go into the Dockerfiles for the consumer service as well as the Eureka 
service, since they look almost the same.
  

### Launching the containers  

Since the orchestration setup on the cluster did no longer work properly, I decided on a minimalistic approach which 
was connecting to every single RaspberryPi node via SSH and manually starting the containers. I think everyone agrees 
that this is not the way things should be. However, I also knew that doing it this way would save me lots of time and 
effort compared to trying to fix the cluster. I hope that I will have the possibility to take my time and fix this in 
the future, since managing containers this way didn't feel really good.
Nevertheless, I went through the following steps for checking my containerized demo application:

 + Starting off with the service discovery, I connected to the Intel NUC via SSH and launched a single Eureka container:
   
   `$ docker run -d -p 8761:8761 localhost:5000/eureka --name eureka`
   
 
 + The next step was starting several instances of my producer service on the RaspberryPi cluster:
 
    `$ docker run -d localhost:5000/weather-producer --name producer`
    
 
 + Finally, I launched as single instance of the consumer service on the NUC:
     
     `$ docker run -d -p 8088:8088 localhost:5000/weather-consumer --name consumer`
     
     
Note that I set up my own Docker Registry on the Intel NUC, that's why the I address the Docker images with 
the "localhost:5000" prefix here.  
   
   
   
### Facing some networking issues
    
As soon as I shipped my services to the cluster for the first time, everything seemed to work well. All containers 
came up, the producer services registered themselves with Eureka and were ready to get looked up by the consumer 
service. So far, so good. But although the consumer app was able to do lookups, no producer instance could be reached
and I got was "connection refused" error messages. Something was definitely wrong.
Shortly afterwards, it began to on me when I recognized that all the producer services running on the Pi cluster 
published a 172.17.0.x IP address to Eureka, although the hosts within were statically assigned an internal IP address 
from the range 141.62.66.x.    
To understand what was happening here, you have to understand how Docker networking works by default: Each host 
running a Docker daemon has a _docker0_ network interface, which in my case had IP 172.17.0.1 (surprise, 
surprise) on each host. By default, every container is bound to that network interface, which allows it to access the
internet, but protects it from being accessed from the outside via any other network interface unless one or more ports 
get explicitly published. This is called "bridge networking". Since every host has its own _docker0_ interface, the 
consumer service on the NUC actually searched for the producer service containers within the NUC's local bridge 
network, instead of reaching out to the public IP addresses of the Pis. So the challenge was finding a way to    
make the producers writing their host's __public__ IP address into the service discovery, which was not so easy 
because the containerized apps had no idea of the host's network interfaces. Everything they saw was _docker0_.  
When discussing that problem with Docker Captain Alex Ellis \(<a href="https://twitter
.com/alexellisuk">@alexellisuk</a>\) at _Docker Distributed Systems Summit_ in Berlin, he told me about a Docker 
networking feature called "overlay network". In short, this allows for defining container networks which can span 
multiple Docker hosts. On top of that, you get DNS for free, meaning that every container within the overlay network can
be reached by a network alias. In case there're several containers with the same alias, you also get fully transparent 
load balancing (to understand load balancing in Docker in depth, 
<a href="http://www.linuxvirtualserver.org/whatis.html">this</a> is a great place to start in my opinion). This 
exactly sounded like the solution to my problem!  
            
    
### Setting up the overlay network   
    
Enabling Multi-host networking for my Docker container setup required a little bit of additional work. The reason for
that was that an overlay network in Docker uses a key-value store under the hood to keep track of the Docker hosts 
participating in the network. Since Docker 1.12, Docker Engine ships with an integrated key-value store, which 
makes managing any additional dependencies (that can possibly fail) obsolete. But since my RaspberryPis ran Docker 1
.10, I could not rely on this new feature and had to setup my own key-value store. In order to get the overlay 
network up and running, I performed the following steps:
         
 + I downloaded and installed (Apache ZooKeeper)[https://zookeeper.apache.org] on the NUC in order to provide the 
 necessary key-value store which for getting started with multi-host networking. By the way, needless to say you can 
 also use a Docker container to do this ;). 
 
 + Then, all the Docker Engines on the Pis and the NUC had to be prepared for multi-host networking. I added several 
 options to the Docker daemon configuration in order to make it join the overlay network by registering with ZooKeeper:
 
        ExecStart=/usr/bin/dockerd -H tcp://127.0.0.1:2375 
        -H unix:///var/run/docker.sock 
        --cluster-store=zk://{NUC_IP}/docker:2181 
        --cluster-advertise=eth0:2375
     
 + Afterwards, I restarted the Docker daemons and checked if they properly registered with ZooKeeper.
         
 + The last step is to create the actual overlay network. Although I also did that on the Intel NUC as the core 
 component of my cluster, this can be done from any node within the network:
           
        $ docker network create --driver overlay --subnet=10.0.9.0/24 my-overlay
    
    
That's it. From then on, I could easily attach my containers to the newly created overlay network when starting them:
 
    $ docker run -d --network my-overlay --network-alias producer localhost:5000/weather-producer`

From the consumer service's perspective, an arbitrary producer microservice could now be reached by searching for a 
"producer" within the overlay network. If you think carefully about that, you might notice that having
Eureka as a service discovery was no longer necessary at this point, since the overlay network already resolved the 
"producer" alias to the IP address wanted and therefore de facto eliminated the need for any external service 
discovery. But again, since I planned to use the Eureka GUI for demo purposes, I decided to keep it and use the 
overlay network for addressing the Eureka server by its container alias "eureka", which spared me the need to hard code 
the IP within the consumer service.  Besides container DNS, the overlay network gave me what was most important to me: 
Valid IP addresses across the whole cluster. So, every app published an IP from the range 10.0.9.x to Eureka after it's 
container had joined the overlay network. In a productive environment, resigning the additional round trip introduced 
by the external service discovery and relying on Docker DNS as well as load balancing instead might be reasonable due 
to performance reasons. 
    
    
<br/>    

## CI workflow

Another aspect of my project was developing a continuous integration workflow that includes automatically 
compiling my services, building runnable JARs, wrap them in Docker images and push them to a image registry. While 
there're many ways and tools to put that into practice, I  elaborated the following approach:
   
   + A Jenkins CI server checks out the source code from Github and triggers a Maven build.
   + At first, Maven compiles the code and packages everything up as a JAR file by means of the 
   [Spring Boot Maven Plugin](http://docs.spring.io/spring-boot/docs/current/reference/html/build-tool-plugins-maven-plugin.html).
   + Subsequently, the [Spotify Docker-Maven-Plugin](https://github.com/spotify/docker-maven-plugin) is used to build
    a Docker image based on a Dockerfile within the project's root directory.
   + After a successful Docker build, the Spotify Docker-Maven Plugin also takes care of pushing the resulting image 
   to a Docker registry.
   
   <div class="image-div"> 
       <img src="{{site.url}}/assets/2016-11-12/ci-workflow.png" class="image-with-caption"/><br/>
       <small class="img-caption"><span>Figure 3: </span>CI tooling and workflow. 
       [Icon sources: <a href="https://jenkins.io/images/226px-Jenkins_logo.svg.png">Jenkins</a>,
       <a href="https://blog.docker.com/media/docker_registry.png">Docker</a>,
       <a href="https://maven.apache.org/images/maven-logo-black-on-white.png">Maven</a>,
       <a href="http://www.ethode.com/dotAsset/a3721082-791e-40a8-88c4-6909d4e114d3.png">Spring Boot</a>,
       <a href="https://image.freepik.com/free-icon/jar-file-format_318-45098.png">JAR</a>]
       </small>
   </div>
   
   
   
### Setting up a private Docker Registry
   
Launching a private registry for Docker images is not very difficult. Docker offers its image registry - how could it
be any different - as a Docker image. The following command is sufficient to pull the latest version of registry v2 
from Docker Hub and launch it on port 5000:
      
      $ docker run -d -p 5000:5000 --name registry registry:2
   
That's exactly what I did on the Intel NUC to configure a private registry for my microservices images.
  
   

### Building an executable JAR with Spring Boot

  
   
   

### Building and pushing Docker images with Spotify Docker-Maven-Plugin

 The Docker-Maven-Plugin by Spotify can be directly configured within the _pom.xml_ file and seamlessly integrates the
 process of building an image and optionally pushing it to Docker Hub or any private registry into a Maven build. 
 Taking the weather consumer service as an example, the plugin configuration may look like this: 
  
  {% highlight xml %}
  <plugin>
    <groupId>com.spotify</groupId>
    <artifactId>docker-maven-plugin</artifactId>
    <version>0.4.11</version>
    <configuration>
        <dockerDirectory>.</dockerDirectory>
        <imageName>localhost:5000/weather-consumer</imageName>
        <imageTags>
            <imageTag>latest</imageTag>
        </imageTags>
    </configuration>
  </plugin>
  {% endhighlight %}
   
   
   There's absolutely no rocket science here. I simply tell the plugin to look for a Dockerfile in the project's root
   directory (we already saw a sample file), define the image name (which is prefixed with the repository location) 
   and tag it as "latest". In order to make the plugin do its work, invoke Maven from CLI along with the following 
   goals specified:
     
        $ mvn docker:build -DpushImage
        
   Note that the _pushImage_ flag is optional and can be omitted if for some reason the resulting image shall not be 
   pushed after the build. 
        
        
<br/>        

## Sources

* Docker, Inc. (2016). _Docker Registry_. [online] Available at: 
[https://docs.docker.com/registry/](https://docs.docker.com/registry/)
[Accessed 12 Dec. 2016]

* Docker, Inc. (2016). _Get started with multi-host networking_. [online] Available at:
  [https://docs.docker.com/engine/userguide/networking/get-started-overlay/](https://docs.docker.com/engine/userguide/networking/get-started-overlay/)
  [Accessed 12 Dec. 2016]

* Netflix, Inc. (2016). _Eureka at a glance_. [online] Available at:
  [https://github.com/Netflix/eureka/wiki/Eureka-at-a-glance](https://github
  .com/Netflix/eureka/wiki/Eureka-at-a-glance)  
  [Accessed 11 Dec. 2016].
  
* Netflix, Inc. (2016). _How to Use_. [online] Available at:
[https://github.com/Netflix/Hystrix/wiki/How-To-Use](https://github.com/Netflix/Hystrix/wiki/How-To-Use)
[Accessed 04 Dec. 2016].

* Netflix, Inc. (2016). _Netflix/eureka_. [online] Available at:
[https://github.com/Netflix/eureka](https://github.com/Netflix/eureka)
[Accessed 05 Dec. 2016].

* Netflix, Inc. (2016). _Netflix/Hystrix_. [online] Available at:
[https://github.com/Netflix/Hystrix](https://github.com/Netflix/Hystrix)
[Accessed 05 Dec. 2016].

* Netflix, Inc. (2016). _Netflix/ribbon_. [online] Available at:
[https://github.com/Netflix/ribbon](https://github.com/Netflix/ribbon)
[Accessed 11 Dec. 2016].

* Pivotal Software. (2016).  _Spring Cloud Netflix_. [online] Available at: 
[http://cloud.spring.io/spring-cloud-netflix/spring-cloud-netflix.html](http://cloud.spring
.io/spring-cloud-netflix/spring-cloud-netflix.html)  
[Accessed 05 Dec. 2016].

