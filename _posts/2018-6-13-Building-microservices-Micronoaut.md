---
layout: post
title: Building microservices with Micronaut 
tags: [ Micronaut, microservices, consul ]
---

I will provide some reasons why is worthy to give it a go to Micronaut in order to implement a microservices.

![_config.yml]({{ site.baseurl }}/images/MICRONAUT-LOGO.png)

# Why another framework?

Let me provide a bit of background first about my thoughts... last year our team splitted a big monolithic JBoss application into 9 or 10 Spring Boot apps, each of them with clear defined responsabilities.
We took the right decision and that helped us dramatically in the transition to a fully Dockerized environment.

At some point we were challenged by our customer to see what would be the impact of implementing a pure microservices architecture.

As our current focus knowledge is mainly around Spring, we started looking to core components from Spring Cloud
and Netflix OSS to allow separately deployed microservices to communicate with each other. If you are not familiar
with this stack I will just say that make development of microservices easier as it brings "out of the box" service discovery,
edge services, gateways, dinamic routing, load balancing, circuit breakers and service registry to start with. 
As usual with all the Spring libraries, a lotof things can be accomplished using auto configuration and convention over configuration.

You will be thinking... what the heck all this rant has to do with Micronaut! I came here to read about this new framework!
Be patient! 

We never had the chance to implement that change, so question is:

If I had to implement now a microservices project will I stick to that decision? using the 'old Sprig Boot ? 
or would I give it a go to the new kid of the block: Micronaut,  which claims to be "natively cloud native" 

# Welcome to Micronaut

Let's see first why I may feel tempted to give it a go and why all the fuzz around.
These are some rough ideas in no particular order of importance. I will let you score which ones you prefer.

* JVM-based architecture, can be run with Java, Kotlin and Groovy.

This is nice although not different from what its Spring competitor provides.

* Natively cloud native.

Instead of adding packages micronaut has been built from the scratch with the cloud in mind.
This area will be the focus of our post. We will see how it supports support for common patterns in microservices like service discovery & registration, circuitbreaker, retries, distributed tracing tools, and support of cloud runtimes, mainly AWS (probably a new post).

* Fast, dazzling fast!  in comparison with Spring.

I was made aware by my workmate  ( hello Jose! ) that Spring 5 have added a significant change on its core container. Before Spring 5 , the candidates component identification for injection was based on classpath scanning. As the number of classes available on the path increases, the start-up time of spring boot apps will increase accordingly. Now Spring 5 builds a file candidate list during compilation time. 
So regardless the number of classes, the time access components is linear.

While this seems a significative improvement, Micronaut goes a step ahead or two.
Annotation metadata is created at compile time, not before. 
There is no usage of reflection at all, however we know Spring uses reflection for nearly everything.
So our performace will not get impacted trying to get configuration data to inject components.
The magic lies on compile time using Groovy AST transformation or AST processors for Java and Kotlin.

What does it mean? With source-level annotation processing one can create source files during the compilation stage. Typical usages of AST processors are creating anotations, that for example change the source code of beans to guarantee inmutability. Another example can be to prevent using the wrong scope modifiers on our variables (i.e force our variables to be declared as final).
With this approach, all the metadata related with our classes is stored and Micronaut avoid usage of reflection on runtime. 

* Automatic Client generation.

On top of the previous improvements and thanks again to AST processer there is super-nice feature.
If you define a server with controllers, there is also the posibility to generate automatically the client code.
How many times do we need to build several endpoints and we end up building a client library based on httpOK, http-apache or RestAssured? Generally tends to be boilerplate code that does not add any business value but takes time, as it needs to be developed and tested.

* Born reactive and non-blocking.

Spring already provides Spring MVC Web Functional Reactive framework using Monos and Flux.
Micronaut comes with the hyper-fast speedy fully Reactive non-blocking compliant servers Netty.
Netty uses NIO to achieve better throughput, lower latency and less resource consumption that tomcat or Jetty.
Using Netty in Spring is possible as web applications built on a Reactive Streams API can be run Netty, Undertow, and Servlet 3.1+ containers.
Being said this, Micronaut brings Netty Out-Of-the-box, so you do not need to do absolutely anything.

* Created by the Graeme Rocher, the guy behind Groovy Grails.

I had the chance to work with Grails in 2008 and from what I remember I see lot of ideas from that old framework that have been migrated from it and are common practices on today's frameworks. So to me this guy is a visionary. He brought things from Ruby on Rails into Groovy. 
Let me describe a few things that were available on Grails 10 YEARS AGO!

    * REPL ( Read-Eval-Print-Loop) has been added recently to Java 9, shame on you!
    * Command line, project scaffolding - now we have Spring Roo, Jhipster
    * Grails Database migrations - now we have Liquibase or Flyway 
    * GORM - Awesome counterpart on the Groovy family with Hibernate.

So in summary if this guy has spent time to sit back and come up with a new framework I will be interested on knowing what is inside. 

I had to deal with some errors in the gitter channel https://gitter.im/micronautfw/questions and want to take this opportunity to thank him , as he was incredible useful and efficiente  whenever I had a question. 

I hope now I have at least get your attention?

# The beers delivery service 

I will use a simple example in order to see whether or not Micronatu helps me to solve the challenges  when building a solution based on microservices. Let's describe our hyper-simple  scenario. 

A bar tender service which is ready to serve beers to multiple customers and get track of the costs to prepare the customer bill once our customer asks for it.

![_config.yml]({{ site.baseurl }}/images/MICRONAUT-BEERS.png)

Let's start with the simplest approach trying to keep a microservices approach in mind.
We will create a project per service so it can be released independently.
Each project will be built using the micronaut framework, exposing endpoint to implement the logic. 
In an ideal world our build process should tag and push an image to Docker so can be deployed effortessly.

In this post we will focus only in the Waiter MicroService and the Billing Service. I will expand on the other services in the next post as I want to explore Distributed tracing and other cool features of the framework.

The scenario could be as follows:

* Different Customers may step in the bar, and ask for a few beers to the Waiter.

* The Waiter needs to report to the  Ticket Billing whenever a customer ask for a beer. Its price is added to the Customer's bill.

* After a few beers, our Customer ask the bill to the Waiter and pay ( leaving a nice tip!).

* Waiter retieves the bill from the Ticket Billing and delivers it to the Customer.

I will highlight some of the interesting bits while developing the service.
The full code can be found on my personal github account

The billing service exposes 3 REST endpoints:

* Add beers to a specific customer bill.

* Reset the information related to a customer bill (useful for testing)

* Retrieve the accumulated cost associated with the number of beers the customer ordered.

```java
@Controller("/billing")
@Validated
public class TicketController {
	
	HashMap<String, Ticket> billsPerCustomer = new HashMap<>();
	
	@Get("/reset/{customerName}")
    public HttpResponse resetCustomerBill(@NotBlank String customerName) {
    	    billsPerCustomer.put(customerName, new Ticket());
    	    return HttpResponse.ok();
    }

    @Post("/addBeer/{customerName}")
    public HttpResponse<BeerItem> addBeerToCustomerBill(@Body BeerItem beer, @NotBlank String customerName) {
    	    Optional<Ticket> t = Optional.ofNullable(billsPerCustomer.get(customerName));
    	    Ticket ticket = t.isPresent() ?  t.get() : new Ticket();
    	    ticket.add(beer);
    	    billsPerCustomer.put(customerName, ticket);
    	    return HttpResponse.ok(beer);
    }
    
    @Get("/bill/{customerName}")
    public Single<Ticket> bill(@NotBlank String customerName) {
    		Optional<Ticket> t = Optional.ofNullable(billsPerCustomer.get(customerName));
    		Ticket ticket = t.isPresent() ?  t.get() : new Ticket();
        return Single.just(ticket);
    }
}
```

In the implmentation you will realize that I'm using a in-memory process Map to store the customer bills. 
We SHOULD be using a proper shared repository (ie, Mongo, KV store, ..whatever), otherwise each service will have its own tracking!!

Also I tried to combine Reactive and non-Reactive endpoints just to compare with Monos and Flux from Spring.

The first attempts spinning our microservice was not succesful. So I added a unit test to reproduce (and hopefully fix the issue). See below

# FEATURE 1: Automatic client generation

I will take this chance to show a cool feature of the framework: automatic client generation
We just need tp create an interface or abstract class with the annotation @Client and define the same methods we declared in our Controller. Micronaut will do the rest and provide an automatically generated Http client ready to be used in our test.

```java
@Client("/billing")
public interface TicketControllerClient {
	
	@Get("/reset/{customerName}")
    public HttpResponse resetCustomerBill(@NotBlank String customerName);

	 @Post("/addBeer/{customerName}")
	 HttpResponse<BeerItem> addBeerToCustomerBill(@Body BeerItem beer, @NotBlank String customerName);
		   

	 @Get("/bill/{customerName}")
	 public Single<Ticket> bill(@NotBlank String customerName);
	    
}
```

Now the client is ready to be used it in our test:

```java
public class TicketControllerTest {
	
	private final String USERNAME="Johh Doe";
	private final String BEER_NAME="mahou5x";
	

    private EmbeddedServer server;
    private TicketControllerClient client;

    @Before
    public void setup() {
        this.server = ApplicationContext.run(EmbeddedServer.class);
        this.client = server.getApplicationContext().getBean(TicketControllerClient.class);
        client.resetCustomerBill(USERNAME);
    }
    
    @Test
    public void shouldAddNewBeer() {
    		BeerItem beerItem = new BeerItem(BEER_NAME, BeerItem.Size.MEDIUM);
    		HttpResponse<BeerItem> response = client.addBeerToCustomerBill(beerItem, USERNAME);
            assertEquals(response.body().getName(), BEER_NAME);
    }
    
    @Test
    public void shouldGetTicketWithZeroWhenCustomerDidNotOrderBeers() {
    		Single<Ticket> response = client.bill(USERNAME);
            assertEquals(response.blockingGet().getCost(), 0,0);
    }

    @After
    public void cleanup() {
        this.server.stop();
    }
}
```

Executing the test fails, as I expected but now I get some debug message that guides me to the resolution of the issue
The beans used by our microservices MUST have a default constructor, otherwise the JSON deserialization will not work.
Once added, the test passes and as consequence our billing microservice  server is able to track tickets related with customer beer orders.

Now our second microservice is the Waiter service, that will expose 2 endpoints.

* Deliver beers to customer.

* Deliver the ticket with the cost to the customer. In order to do that it will contact to the Billing service, using exactly the same client we used before when testing the billing service.

```java
@Controller("/waiter")
@Validated
public class WaiterController {

    @Inject
    TicketControllerClient ticketControllerClient;

    @Get("/beer/{customerName}")
    public Single<Beer> serveBeerToCustomer(@NotBlank String customerName) {
        return Single.just(new Beer("mahou5x", Beer.Size.MEDIUM));
    }
    
    @Get("/bill/{customerName}")
    public Single<CustomerBill> bill(@NotBlank String customerName) {
        Single<Ticket> ticket = ticketControllerClient.bill(customerName);
        return Single.just(new CustomerBill(ticket.blockingGet().getCost()));
    }
}
```

So now we have the core of the behaviour, we will start seeing how Micronaut can help us in a microservices environment.
Our example allows direct communication between Waiter and Billing because Waiter is aware of the coordinates (server and port) where the service is running. However when you start adding more microservices the problems gets more complex. 

# FEATURE 2: Service Discovery and Registration

Imagine we have multiple Waiters and a single instance of the Ticket Billing service cannot cope with the demand to track Customer tickets. Therefore we need to spin up more instances of the service. But how will the Waiter service know which instance of the billing service needs to get in touch with?

The recommended approach to solve that is use Service Discovery pattern.

Traditional approaches are to use a well known address and resolve that via DNS (not really good for latency and propagation).

Other alternatives imply add a load balancer between services so our service only is aware of the load balancer IP and let the load balancer do the rest. This approach is valid if we know beforehand the number of instances the load balancer will proxy, but in a microservices environment the number of instances of services are scaling up and downn, and the configuration maybe pointing to an instance that is already dead. In order to overcome that the trend is to add sidecar applications together with our microservices that modify in runtime the configuration of our load balancer.

There is a better solutions which imply that each microservice register itself on startup against a Services Registry. Microservices that need to use that service, will "discover" the service contacting the service registry and resolving its ip by service name. The only drawback is that we need to modify the microservice itself to be aware of the services registry.

And this is where Micronaut gets handy. With minimum configuration you can achieve that in a snap.

We will go through the steps to register our Billing Service:

Our billing microservice need dinamic port allocation instead of fixed port to avoid collisions on multiple startup.
Commenting the port setting in our configuration file is enough

```yaml
#micronaut.server.port=8081
```

Also our Billing service needs to register itself. We just need to enable it and define where our registry service is listening.
Micronaut supports Eureka,Consul and Kibernetes just by adding a new line!
We will use Consul. Be sure your application.properties contains the following configuration

```yaml
consul:
  client:
    registration:
      enabled: true
    defaultZone: "${CONSUL_HOST:localhost}:${CONSUL_PORT:8500}"
```

Do not forget to add the dependency that enables service registration in your Maven POM file.

```xml
        <dependency>
          <groupId>io.micronaut</groupId>
          <artifactId>discovery-client</artifactId>
        </dependency>
```

The only thing we need to do is add a reference in our client annotation to the service name. 
In our application.properties we named the BillingService as billing. Our client annotation now has the identifier

```java
@Client(id="billing")
public interface TicketControllerClient {
	.... 
    //identical to the method we had before
	....
}
```

Let's see it in action. We will start a Consul instance using public docker image:

```java
docker run -p 8500:8500 consul
```

The modification breaks our initial integration test. After scratching my head, I realize that there is a period of time since the service is registered in consul and Consul updates the status to consider the service healthy. As a consequence I had to modify our test slightly to add a small delay of 1 second.

```java
    @Client(id="billing")
    @Inject
    private TicketControllerClient client;

    @Before
    public void setup() {
        this.server = ApplicationContext.run(EmbeddedServer.class);
        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        this.client = server.getApplicationContext().getBean(TicketControllerClient.class);
        client.resetCustomerBill(USERNAME);
    }
```

Once the test if fixed, we can start 2 or 3 instances of billing services. 
Inspecting the log files we can see that the service is talking with Consul, YAY!

```yaml
08:54:15.291 [main] INFO  io.micronaut.runtime.Micronaut - Startup completed in 1574ms. Server Running: http://localhost:63510
08:54:15.336 [nioEventLoopGroup-1-3] DEBUG io.netty.buffer.AbstractByteBuf - -Dio.netty.buffer.bytebuf.checkAccessible: true
08:54:15.337 [nioEventLoopGroup-1-3] DEBUG i.n.util.ResourceLeakDetectorFactory - Loaded default ResourceLeakDetector: io.netty.util.ResourceLeakDetector@6562dbf0
08:54:15.488 [nioEventLoopGroup-1-3] DEBUG i.m.http.client.DefaultHttpClient - Sending HTTP Request: PUT /v1/agent/service/register
08:54:15.488 [nioEventLoopGroup-1-3] DEBUG i.m.http.client.DefaultHttpClient - Chosen Server: localhost(8500)
```

Now we head to Consul UI, we will see a entry for Billing service with 3 instances and its check health.

![_config.yml]({{ site.baseurl }}/images/MICRONAUT-CONSUL.png)

So far so good.. but this is not very useful unless our Waiter service can figure out the address of one of the instances. Now we will step through the modifications to our Waiter Service so it can "see" the Billing service:

The remaining steps are nearly identical to the ones we did when configuring Billing services. 

Our _application.properties_ files needs to be aware of the existence of a Consul server. 
We will also have registration enabled so Consul keeps track of every service in our solution.
We need to add the same dependency to our Waiter Maven POM file. 
However in this case we will use fixed port allocation so we can hit our Waiter service directly.

To be 100% sure the service discovery works we modify again slightly our domain model and our /bill/{customerName} method. Our ticket holds now metadata simulating the desk identifier. 
For simplicity  we will use the port associated to the server 
See below an extract of the changes required on the TicketController.

```java
@Controller("/billing")
@Validated
public class TicketController {

	final EmbeddedServer embeddedServer;

	public TicketController(EmbeddedServer embeddedServer) {
		this.embeddedServer = embeddedServer;
	}
	
    //Other methods remain identical
    //
    
    @Get("/bill/{customerName}")
    public Single<Ticket> bill(@NotBlank String customerName) {
    		Optional<Ticket> t = Optional.ofNullable(billsPerCustomer.get(customerName));
    		Ticket ticket = t.isPresent() ?  t.get() : new Ticket();
    		ticket.setDeskId(embeddedServer.getPort());
        return Single.just(ticket);
    }
}
```

The change needs to be propagated to the consumer of the service. The Waiter service just return that metadata 

Theoretically if everything works whenever we hit our Waiter service several times we will get different counter id. Micronaut uses _round-robing load balancing_ on the client side, so if we have 3 services I expect the desk identifiers to rotate evenly distributed among the 3 instances.

```bash
cd ~/Documents/workspace/micronaut-examples/beer-billing
for i in `seq 1 9`;do
                echo "\nAttempt $i\n"
                curl "http://localhost:8082/waiter/bill/mycustomer"
done

Attempt 1

{"cost":0.0,"deskId":37522}
Attempt 2

{"cost":0.0,"deskId":48465}
Attempt 3

{"cost":0.0,"deskId":19508}
Attempt 4

{"cost":0.0,"deskId":37522}
Attempt 5

{"cost":0.0,"deskId":48465}
Attempt 6

{"cost":0.0,"deskId":19508}
Attempt 7

{"cost":0.0,"deskId":37522}
Attempt 8

{"cost":0.0,"deskId":48465}
Attempt 9

{"cost":0.0,"deskId":19508}%
```

# FEATURE 4: Retries

Let's face it.. life is not perfect. And sometimes sh*t happens (excuse my english). 

We are thinking on an ideal world where the Waiter gets assigned one of the 3 available desk (Ticket Billing service). If one fails, other would be available right?

Let's constraint a bit more our scenario. A single Waiter with a single Desk assignment... 
To make things worse the behaviour of the Desk for Billing is intermitently wrong.. Sometimes does not accept any new request. And in the worst scenario it becomes totally iresponsive. What could our poor waiter do in that case?

Now imagine a complex microservice architecture where you have many services communicating with each other, in different networks and constrained by cpu limits and memory constraint. Obviusly the clever guys have already tried to solve that problem before and luckily enough for us Micronaut has incorporated in a way which is not disruptive in our code.

Let's get back to our Beer bar. We left our Waiter "in despair" because the existing issues across all the Ticker Billing terminals.

There are a couple of things he could do. He could keep trying just in case the Desk get back to normal a few time, as the nature of the error was intermitent ... and hopefully once of that tries he will succeed.

The only thing we need to do is provide that behavior to our client.
using @Retryable annotations on our @Client

```java
@Client(id="billing", path="/billing")
@Retryable(attempts = "10", delay = "2s")
public interface TicketControllerClient {
    ......
    .......
```

After starting both services at the same time, we will simulate a failure bringing down Billing Service to see if it works.
And voila!

```yaml
09:25:26.616 [nioEventLoopGroup-1-22] DEBUG i.m.r.i.DefaultRetryInterceptor - Retrying execution for method [Single bill(String customerName)] after delay of 2000ms for exception: Connect Error: Connection refused: localhost/127.0.0.1:2127
09:25:26.618 [nioEventLoopGroup-1-22] DEBUG i.m.context.DefaultBeanContext - Resolving beans for type: <RetryEvent> io.micronaut.context.event.ApplicationEventListener 
09:25:26.618 [nioEventLoopGroup-1-22] DEBUG i.m.r.i.DefaultRetryInterceptor - Retrying execution for method [Single bill(String customerName)] after delay of 3000ms for exception: Only one subscriber allowed
09:25:26.618 [nioEventLoopGroup-1-22] DEBUG i.m.context.DefaultBeanContext - Resolving beans for type: <RetryEvent> io.micronaut.context.event.ApplicationEventListener 
09:25:26.618 [nioEventLoopGroup-1-22] DEBUG i.m.r.i.DefaultRetryInterceptor - Retrying execution for method [Single bill(String customerName)] after delay of 4000ms for exception: Only one subscriber allowed
09:25:26.618 [nioEventLoopGroup-1-22] DEBUG i.m.r.i.DefaultRetryInterceptor - Cannot retry anymore. Rethrowing original exception for method: Single bill(String customerName)
```

I noticed weird behaviour with timestemps as I would expect certain cadence in the logs, a message every second instead of all of them in one go. After speaking with Graeme Roecher, he told me that the way Retry works is slightly different when your endpoints are reactive. I'm glad I helped some how to diagnose the issue ;) and he fixed very promptly <https://github.com/micronaut-projects/micronaut-core/commit/eb7dd1f274de88ab874c2a70ba1fba6e4bca565e> 

It is also worth mentioning that on a scenario where we start ONLY Waiter service, when we make a request @Retryable does not gets triggered because apparently the first thing it tries is do a service lookup against Consul, and throws a Service not available exception. Also Graeme suggested that probably the behavior of lookup could be modified if the Retry annotation is present. It would be useful on scenarios where one starts up the whole microservices, and some services depend on others.

If we restart the Ticket Billing service instance, our Waiter will recover automatically and connect to the Billing service.

# FEATURE 5: Circuit Breaker

Whenever there is communication between two components there is always a chance for things go wrong.
What shall we do when there is a failure?  Do we fail silently sending to a message queue to be processed asyncronously later?
Do we reroute our request to a different service? ... or do we return mock data to avoid a crash in our system?

Also is very likely that when a system is down, it will take a while to spin up again. Do we want to keep trying till the system is back? or maybe is a better option just to chose an alternative option after a maximum number of attempts. In a peak period a system "in pain" could also benefit from getting less requests instead of endless requests that do not help to recover. 

So in microservices there is a well known pattern to help with these scenarios. The Circuit Breaker pattern.
The idea behind means adding a "vigilante" in the communication between 2 components to get track of failures.
Initially the circuit is in a "opened" state. Every time there is a failure a counter is increased till it reachs a threshold.From that moment onwards the circuit is "Closed" and replies immediately with an error avoiding unnecesary timeouts. The Circuit Breaker may have some polling component in order to restore the state to Open once the communication is restablished or simply rely on timers.

Micronaut allows with a single line define the following behavior.
If a service fails, do 5 attempts, waiting 5 seconds the first attempt, 10 seconds the second attempt and so on using the multiplier property of the annotation. Also we can instruct to reopen the circuit after 5 minutes.

```java
@Client(id="billing", path="/billing")
@CircuitBreaker(delay = "5s", attempts = "5", multiplier = "3", reset = "300s", excludes = NonValidBeerException.class)
public interface TicketControllerClient {
    ......
    .......
```

The only thing that is worthy mentioning is that we also can reroute to a default implementation.
In this case when combined with Retrayable, this client will be chosen once the retries go over the threshold. 

```java
@Fallback
public class NoCostTicket implements TicketControllerClient{
    @Override
    public HttpResponse<BeerItem> addBeerToCustomerBill(BeerItem beer, @NotBlank String customerName) {
        return HttpResponse.ok();
    }

    @Override
    public Single<Ticket> bill(@NotBlank String customerName) {
        return Single.just(new Ticket());
    }

    @Override
    public HttpResponse resetCustomerBill(@NotBlank String customerName) {
        return HttpResponse.ok();
    }
}
```

I think that is it for now. Overall I scratched just the surface but I see a powerful set of features that would make development of cross-concerns related to microservices a breeze to play with.

In next installment of Micronaut series I plan to explore other interesting concepts such us reactive endpoints, distributed configuration, distributed tracing and integration with AWS serverless functions.



