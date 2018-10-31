---
layout: post
title: Building Micronaut serverless AWS Functions and integration with microservices
tags: [ micronaut, java, microservices, aws ]
---

The following post would explore how Micronaut enables developement of serverless functions and integration with AWS Lambda as cloud providers.

![_config.yml]({{ site.baseurl }}/images/MICRONAUT-LAMBDA-FINAL.png)

# Previous articles

In previous entries we have seen the benefits of using static compilation 
and how fast our microservices start in comparison with Spring. 

We also saw how micronaut provides out of the box [microservice cross concerns like service discovery, distributed tracing, circuit breakers and retries][1]. 

We explored how easy was to provide [distributed configuration, implement server side events][2] and finally we [deployed everything into a Kubernetes cluster][3] 

# What will be build?

In our starting point scenario, the cost of the ticket was part of the billing service itself. It was implemented via the class **BeerCostCalculator**. I will use that class as a candidate to be moved as a Function. 

We will decouple that calculation logic as a serverless function running in AWS.

![_config.yml]({{ site.baseurl }}/images/MICRONAUT-LAMBDA-DIAGRAM.png)

# Upgrading to Micronaut 1.0.0 and preparation steps

I heard that Micronaut released GA (1.0.0) [release][4], so I upgraded all my Beer services 
to use the right dependencies. 

Be aware that in the new release Micronaut has renamed all its internal dependencies and bom files by adding a prefix "micronaut-".

I used **sdkman** to be sure I had the right Micronaut version and because I want to use its "mn" client that help us creates templated projects, in my case a function. 

```bash
$ curl -s "https://get.sdkman.io" | bash
$ source "$HOME/.sdkman/bin/sdkman-init.sh"
$ sdk install micronaut 1.0.0
Downloading: micronaut 1.0.0
In progress...
######################################################################## 100,0%
Installing: micronaut 1.0.0
Done installing!
Setting micronaut 1.0.0 as default.
```

I found some issues while upgrading to 1.0.0 related with duplicated bean candidates

The Micronaut guys at Gitter channel were helpful as usual and more specifically @jameskleeh volunteered to detect what was going on. 

Eventually everythig was releated to the maven-shade-plugin and how the ResourceTransformer class duplicated jar entries due to transitive dependencies.
He also highlighted that if waiter depended fully on billing jar, I was exposing the billing endpoints through my waiter microservice! i.e our user could go to the till without even asking our poor waiter.

So I had to refactor the dependencies so is a cleaner architecture. Basically added a couple of modules. One containing only the *@Client* interface and the *@Fallback* so they could be used both by the unit test and by our waiter. I also shared the model across projects.

+ billing > billing-client (for unit testing) > model
+ waiter > billing-client > model.

# Creating a function archetype with Micronaut

We will create a new project to declare our function so it has its own release cycle and can be changed independently from the other microservices. Let's imagine that the cost will be based on a simple formula based on brand name, applying a multiplying factor when we deal with pints of beer, so they will
be more expensive.

I created a new project "beer-cost-function-app" from the scratch using the Micronaut client 

```bash
$ mn create-function beer-cost-function-app
```

Although I could use the gradle build, I'm more a maven guy so I mavenized the project using as reference my pom files from beer-billing and beer-waiter. 


The interesting bits in your pom to enable functions are these 2 dependencies
```xml
		<dependency>
			<groupId>io.micronaut</groupId>
			<artifactId>micronaut-function-client</artifactId>
			<version>${micronaut.version}</version>
			<scope>compile</scope>
		</dependency>
		<dependency>
			<groupId>io.micronaut</groupId>
			<artifactId>micronaut-function-aws</artifactId>
			<version>${micronaut.version}</version>
			<scope>compile</scope>
		</dependency>
```

I splitted responsabilities across different projects: 

+ client (beer-cost-function-client)
+ model(beer-cost-function-model) 
+ server(beer-cost-function-app) 


# Moving to Micronaut Functions

Micronaut strives for simplicity and the only thing we have to do is add a annotation to our class
and guarantee that our class implements any of the functional interfaces available in the package *java.util.function*:

Depending on the nature of our calculation we could chose from :

+ *Supplier*  -  Returns a result, no need of input parameters
+ *Function* and *Bifunction* - Returns a result recieve one or two input parameters
+ *Consumer* and *BiConsumer* - Performs a operation over one or two input paramteters, however does not return anything.

In our case we will chose to implement Function that will receive a request containing the beers and it will
return the cost associated.

```java
package micronaut.demo.beer.function;

import java.util.HashMap;
import java.util.Map;
import java.util.function.*;
import io.micronaut.function.FunctionBean;

@FunctionBean("beer-cost")
public class CostCalculator implements  Function<TicketCostRequest, TicketCostResponse>{

    private static final double SMALL_FACTOR=1;
    private static final double PINT_FACTOR=1.8;

    private static final Map<String, Double> beerCost = new HashMap<String, Double>();
    static {
        beerCost.put("FREE-BEER", 0.0);
        beerCost.put("MAHOU", 1.5);
        beerCost.put("HEINEKEN", 2.0);
        beerCost.put("FRANCISKANER", 2.5);
        beerCost.put("PAULANER", 2.8);
    }

    @Override
    public TicketCostResponse apply(TicketCostRequest ticketCostRequest) {
        return new TicketCostResponse( allBeersCost(ticketCostRequest));
    }

    private double allBeersCost(TicketCostRequest ticketCostRequest) {
        return ticketCostRequest
                .getBeerItems()
                .stream()
                .map( beer ->  calculateBeerCost(beer))
                .mapToDouble(i->i).sum();
    }

    private double calculateBeerCost(TicketBeerItem beer) {
        switch (beer.getSize()) {
            case "S" : return SMALL_FACTOR* beerCost.get(beer.getName());
            case "P":  return PINT_FACTOR* beerCost.get(beer.getName());
            default: return 0;
        }
    }
}
```


# Testing your function 

We could test our function by deploying into AWS, trigger an execution and see how it works.
But really our class is just a function that can be fully unit tested.

Still Micronaut has a few goodies that make our live easier.

If you remember on our first post I went through the awesome Automatic Client generation feature.

We just created an interface or abstract class with the annotation @Client and defining exactly the same methods as our Controller, we had for free a client that can be used for testing or for interaction among
our services. 

Following the same paradigm we can create a client function doing exactly the same, just with a different 
annotation 

```java
package micronaut.demo.beer.function;

import javax.inject.Named;

import io.micronaut.function.client.FunctionClient;
import io.micronaut.http.annotation.Body;
import io.reactivex.Single;

@FunctionClient
public interface ClientCostCalculator {
	 @Named("beer-cost")
	 public Single<TicketCostResponse> apply(@Body TicketCostRequest ticketCostRequest) ;
}
```

This is the simplest client to our function. 

If we think in AWS Lambda we still may take benefit on some of Micronaut features. We will see later while testing how Lambda execution can take a while to warm up... so is never a bad idea to make our client calls more robuts with a retry policy.

And improved version (ideally configurable externally) could be 

```
@FunctionClient
public interface ClientCostCalculator {
	 @Named("beer-cost")
     @Retryable(attempts = "3", delay = "2s") 
	 public Single<TicketCostResponse> apply(@Body TicketCostRequest ticketCostRequest) ;
}
```

Our Function test is again straight forward:

```java
public class ClientCostCalculatorTest {
	
	@Test
	public void testBeerCost() throws Exception {
        EmbeddedServer server = ApplicationContext.run(EmbeddedServer.class);
        ClientCostCalculator client = server.getApplicationContext().getBean(ClientCostCalculator.class);
        TicketBeerItem beer1 = new TicketBeerItem("MAHOU", "S");
        TicketCostRequest request = new TicketCostRequest(Arrays.asList(beer1));
        TicketCostResponse responseCost = client.apply(request).blockingGet();
        assertEquals(1.5, responseCost.getCost(),0);
        server.stop();
    }
}
```

One very important warning for those like me who dare to implement their own Micronaut functions using Inmutable POJO objects. 

I scratched my head for a long time till I found the source of weird behaviour when working with inmutable POJOS, i.e without setters and always prefering usage of constructors with parameters. 

So there are two possible approaches. 

+ The initial one was kind-of-obviuos, just adding default constructor both in your request (TicketCostRequest) and response (TicketCostRequest) objects. Once added the test worked like a charm.

+ Graeme Rocher was helpful as usual in the Gitter channel, kudos to you! . He reminded me that in case I want to use Inmutable objects we need to add  @JsonCreator to map the constructor. So eventually I opted by that approach. See an example below on how my Request POJO looks like.

```java
public class TicketCostRequest implements Serializable {

	private static final long serialVersionUID = -3999476323141649992L;
	private List<TicketBeerItem> beerItems = new ArrayList<>();

    @JsonCreator
    public TicketCostRequest(@JsonProperty("beerItems") List<TicketBeerItem> beerItems) {
        this.beerItems = beerItems;
    }

    public List<TicketBeerItem> getBeerItems() {
        return beerItems;
    }
}
```

# Exposing your function as REST endpoint

Micronaut provides an easy way to expose our functions as REST endpoints. 

Be sure your pom file contains
```xml
		<dependency>
			<groupId>io.micronaut</groupId>
			<artifactId>micronaut-function-web</artifactId>
			<version>${micronaut.version}</version>
			<scope>compile</scope>
		</dependency>
```

So if you have coded a Function

+ Functional interfaces that require input parameters are converted into POST endpoint ready to accept 
payloads.
+ Functional interfaces that do not require input parameters like Supplier are converted into GET endpoint.

The name of the endpoint matches with the name defined in our functional bean, i.e "beer-cost" in our case.

So theoretically we could send POST request to our /beer-cost endpoint containing a JSON payload  like this one.

```json
{"beerItems" : [
    {"name":"MAHOU", "size":"S"},
    {"name":"HEINEKEN", "size":"P"}]}
```

You can start directly our server

```bash
mvn exec:exec -Dexec.mainClass="micronaut.demo.beer.function.CostApplication"
```

And on another terminal session 

```bash
curl -d ' {"beerItems" : [{"name":"MAHOU", "size":"S"},{"name":"HEINEKEN", "size":"P"}]}' -H "Content-Type: application/json" -X POST http://localhost:8080/beer-cost
##response
{"cost":5.1}
```

# Deploying to AWS

Our last step will be deploying our function into AWS. This is why we are here, right?

For this you will need to have an account in AWS, otherwise you will not be able
to complete the steps.

We are already generating a shadow jar using maven-shade-plugin (see pom.xml)

```bash
$ mvn package
#Generate shaded jar at $HOME/micronaut-ms/beer-cost-function-app/target/cost-app-0.0.1-SNAPSHOT-shaded.jar
```

## Configure your lambda

If this were a real project with more AWS resources I would link my build process with a step using CloudFormation or Terraform to automate the creation of resources in AWS but for this example I will just create the AWS Lambda manually. 

Chose a region close to your location, in my case Paris is the closest to Malaga ;)

Just create a new function with a name "beer-cost" with Java 8 runtime and creating a "basic_lambda_execution" role. 



Just upload the file $HOME/micronaut-ms/beer-cost-function-app/target/cost-app-0.0.1-SNAPSHOT-shaded.jar and add as Handler the following value: io.micronaut.function.aws.MicronautRequestStreamHandler

## Configure a test event

We will use the same one we used for testing locally 

![_config.yml]({{ site.baseurl }}/images/MICRONAUT-LAMBDA-TEST.png)

## Execute your lambda

First execution took more than 20s, which is the usual warm up period in AWS Lambda but after that subsequent executions where on average around 1sec, which shows why using Micronaut is an excelent option to implement serverless functions.

![_config.yml]({{ site.baseurl }}/images/MICRONAUT-LAMBDA-EXEC.png)

You can also see that output logs are available in CloudWatch 

![_config.yml]({{ site.baseurl }}/images/MICRONAUT-LAMBDA-CLOUDWATCH.png)

# Remote execution of functions

In previous sections I showed how we we could use ClientFunction as part of our unit tests in order to call our function. That is great, but now what we really need is to integrate our billing service in a way that costs calculation is performed by our  function. 

We will add a new implementation of 

```java
public interface CostCalculator {
    public double calculateCost(Ticket ticket) ;
}
```

The new implementation will use the FunctionClient  ClientCostCalculator to interact with our remote Function deployed in AWS. See below

```java
@Primary
public class RemoteFunctionBeerCostCalculator implements TicketCostCalculator {
	
	private  ClientCostCalculator client;
	
	@Inject
	public  RemoteFunctionBeerCostCalculator (ClientCostCalculator client) {
		this.client = client;
	}

	public double calculateCost(Ticket ticket) {
		
		List<TicketBeerItem> beerItems = new ArrayList<>();
		ticket.getBeerItems().forEach(beerItem->beerItems.add(map(beerItem)));
		
		TicketCostRequest ticketCostRequest = new TicketCostRequest(beerItems);
		return client.apply(ticketCostRequest).blockingGet().getCost();
	}

	private TicketBeerItem map(BeerItem beerItem) {
		return new TicketBeerItem(beerItem.getName(), map(beerItem.getSize()));
	}

	private String map(Size size) {
		if (size.equals(Size.SMALL)) {
			return "S";
		} else {
			return "P";
		}
	}
}
```


Note that we have added @Primary because we have 2 implementations of the interface ( the old one we tried to get rid of) and the new one. That helps Micronanut to chose which one to use.

Tests work fine again, but we are still hitting our function using the web interface exposed by our billing app.

# Configure for remote execution

Now the only thing remaining is add a configuration file so Micronaut is aware that the function should be invoked remotely in our cloud provider. 

# A final note on dependencies

Let's revisit again how our components & dependencies look like

![_config.yml]({{ site.baseurl }}/images/MICRONAUT-LAMBDA-FINAL.png)

The most tedious work was splitting in subprojects and separate client from server as a good practice. So for example now beer-billing only depends on the client side of functions and controllers.  If we do not do that, we could be exposing unwanted endpoints as we explained at the very beginning.

It's very important to highlight that clients (both for functions and for controllers) must not use the shade plugin. Otherwise we will have duplicated bean candidates and Micronaut will not be able to assign one. 


# Useful links

+ [Micronaut 1.0.0.RC3 release notes][4]
+ [Source code in github][5]
+ [Micronaut series - part I][1]
+ [Micronaut series - part II][2]
+ [Micronaut series - part III][3]


[1]: https://mfarache.github.io/mfarache/Building-microservices-Micronoaut/
[2]: https://mfarache.github.io/mfarache/Traceability-microservices-Micronoaut/
[3]: https://mfarache.github.io/mfarache/Deploying-Micronaut-Kubernetes/
[4]: https://docs.micronaut.io/1.0.0.RC3/guide/index.html#whatsNew
[5]: https://github.com/mfarache/micronaut-ms













