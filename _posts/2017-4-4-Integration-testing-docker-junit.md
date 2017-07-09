---
layout: post
title: Run integration tests using docker and junit
tags:   [ integration testing, docker, junit ]
---

In this post I will share the steps that I followed to achieve integration testing that can be launched from your favourite IDE or run locally or in a CI server via maven using palantir Docker compose junit rules.

# Project background context

In our current project we have pretty decent acceptance testing based on Arquillian. However these acceptance tests are not really a replica of how the artifacts are deployed into production and other stages.

In our customer environments (DEV, QA, TRN, PRD) we use docker and we have about 11 containers, some of them are just Spring boot fat jars, others are mule ESB instances and others are Tomcat web apps. But our setting in Arquillian are defined to deploy those artifacts as war files on Jetty/Tomcat web applications servers. We also use an in-memory database that is initialized and loaded with data before the execution of our tests begin.
Its far from ideal as some native ORACLE queries just do not work on HQLDB so we need to tweak things to make it work.

We also have automated tests using Ruby/Cucumber/Gherkin spec that allow us to apply definition of user readable stories.

However our ESB layer (Mule) is craving for improvements! We tried with no luck adding a suite to be part of the Arquillian acceptance test project but was not trivial. We have been using Munit for functional testing but we feel it was not enough as we were missing real integration testing.


# Integration test proof of concept scenario

Our integration testing scenario can be summarized as follows:

+ Third party application ORIGIN push XML messages to a RabbitMQ
+ Our Mule ESB layer handle those XML messages, doing validation steps, enriching the information provided by another application
+ ENRICHER, transformation of payload and eventually a message is sent to an application DESTINY

This diagram "hyper-simplified" explains the overall flow. Literally tons of things happen in our ESB layer but this is out of scope within this post.

![_config.yml]({{ site.baseurl }}/images/ESB Enrichment Flow.png)

Really I was interested in verifying the behaviour of the ESB to be sure that the output messages sent to DESTINY matches with our expectations based on the message sent by the application ORIGIN.

Before we go ahead let me add some context about our components

+ We are not responsible of ORIGIN nor ENRICHER so those components are perfect candidates to be mocked using Wiremock.
WireMock is a simulator for HTTP-based APIs. Some might consider it a service virtualization tool or a mock server.
+ RabbitMQ must be a real server, as this is our entry point for all the scenario variations.
+ DESTINY is our application .... and yes we have it dockerized .... but relies on another ORACLE container and multiple dataset that must be loaded into the database before test start. As consequence I thought that would be overkilling. I just need to know that messages arrived to DESTINY, what happens next is not really our concern because we have an awesome acceptance testing on DESTINY covering nearly 95% of the code base. Someone may argue this is not integration testing and they can be right but as I explained my target was to test the behaviour of Mule as a black box.

So I hacked quickly a Spring Boot application  that allows me to store each request in a in-memory Map , so I can query for them within my Junit test. Is dirty and nasty I know ... but I could not think anything more simple than that ;).

<script src="https://gist.github.com/mfarache/dddc21330a1a602931f9c788c03220e9.js"></script>  

Would not be great if I can spin up some containers in one go to simulate the scenario?
What if we can launch that using old-fashioned Junit testing so I can write my integration testing in plain JAVA?

The first question can be easily solved with docker compose. If you version your compose file together with the source code it guarantees that anyone in the team can run the stack locally. With the right settings and VPN I could potentially can start up a QA environment or even a production stack just as easy as running one locally.

What about the second question ? Mr Google to the rescue!. Obviously Im not the first one facing that questions so I evaluated two possible options:

- [Arquillian Cube][1] : An Arquillian extension that can be used to manager Docker containers from Arquillian. With this extension we can start a Docker container with a server installed, deploy the required deployable file within it and execute Arquillian tests. Although it sound like a great fit for our project I felt adventorous and wanted to try something new and not coupled to Arquillian.

- [Awesome Palantir Junit rule][2]
I felt in love when I found the simplicity of the approach. Just a plain Junit rule with many extension points that allow you to configure mainly which is the set of containers you want to spin up before your tests are run.

The project is beatifully coded, you can see someone has put a very decent effort thinking how to do it right.

So let´s get our hands dirty to set up a basic project

## Add palantir dependencies to your maven pom.xml
You can focus on the palantir dependencies only, the rest are related to junit, harmcrest matchers and REST assured that was my preferred approach to test the interactions within the components


```xml
<dependencies>
  <dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-lang3</artifactId>
    <version>3.5</version>
  </dependency>
  <dependency>
    <groupId>ch.qos.logback</groupId>
    <artifactId>logback-classic</artifactId>
    <version>1.1.10</version>
  </dependency>
  <dependency>
    <groupId>org.hamcrest</groupId>
    <artifactId>hamcrest-all</artifactId>
    <version>1.3</version>
    <scope>test</scope>
  </dependency>
  <dependency>
    <groupId>com.google.code.gson</groupId>
    <artifactId>gson</artifactId>
    <version>2.6.2</version>
    <scope>test</scope>
  </dependency>
  <dependency>
    <groupId>com.jayway.restassured</groupId>
    <artifactId>rest-assured</artifactId>
    <version>2.9.0</version>
    <scope>test</scope>
  </dependency>
  <dependency>
    <groupId>com.rabbitmq</groupId>
    <artifactId>amqp-client</artifactId>
    <version>4.1.0</version>
  </dependency>
  <dependency>
    <groupId>junit</groupId>
    <artifactId>junit</artifactId>
    <version>4.12</version>
    <scope>test</scope>
  </dependency>
  <dependency>
    <groupId>org.assertj</groupId>
    <artifactId>assertj-core</artifactId>
    <version>3.6.2</version>
    <scope>test</scope>
  </dependency>
  <dependency>
    <groupId>com.palantir.docker.compose</groupId>
    <artifactId>docker-compose-rule-junit4</artifactId>
    <version>0.31.1</version>
    <scope>test</scope>
  </dependency>
</dependencies>

```

## Write your unit tests

Before we dwelve into the test itself, let me explain a couple of helper classes I wrote in order to simplify scenario reuse across different tests. The first one <DockerScenario>  is just a wrapper I created so I can easily create different scenarios that could be reused by different tests. Note there is a list of named services . Remember this when speak about the docker-compose file. You will see there is match per service name.

```java
package com.mycompany.integration.mule.scenario;

import java.util.ArrayList;
import java.util.List;

import com.mycompany.integration.base.DockerInitializer;
import com.mycompany.integration.utils.SleepUtils;
import com.palantir.docker.compose.DockerComposeRule;

public class DockerScenario {

    private static List<DockerServices> getServices() {
        List<DockerServices> services = new ArrayList<DockerServices>();
        services.add("DESTINY");
        services.add("MULE");
        services.add("WIREMOCK");
        services.add("RABBITMQ");
        return services;
    }

    public static DockerComposeRule getDockerScenarioMuleTesting() {
        // allow some time before starting, in case there was a previous test
        // executing
        SleepUtils.sleepSeconds(10);
        return new DockerInitializer("src/test/resources/docker-compose.yml",
                getServices()).getRule();
    }

}
```

Lets have a look to a base class where the magic happens, before we move into the  scenario itself

```java
package com.mycompany.integration.base;

import java.util.List;

import org.joda.time.Duration;

import com.mycompany.integration.mule.scenario.DockerServices;
import com.palantir.docker.compose.DockerComposeRule;
import com.palantir.docker.compose.DockerComposeRule.Builder;
import com.palantir.docker.compose.connection.waiting.HealthChecks;

public class DockerInitializer {

    private String dockerComposeFile;
    private List<DockerServices> services;

    public DockerInitializer(String dockerComposeFile, List<DockerServices> services) {
        this.dockerComposeFile = dockerComposeFile;
        this.services = services;
    }

    public DockerInitializer(String dockerComposeFile) {
        this.dockerComposeFile = dockerComposeFile;
    }

    public DockerComposeRule getRule() {
        Builder builder = DockerComposeRule.builder().file(dockerComposeFile);
        for (DockerServices dockerServices : services) {
            builder.waitingForService(dockerServices.getServiceName(),
                    HealthChecks.toHaveAllPortsOpen(),
                    Duration.standardMinutes(3));

        }
        return builder.saveLogsTo("./docker-logs").build();
    }

}

```
This class allows to specify a Docker compose file and a set of services to be monitored before the test start.
Please note that Palantir only waits for each container service to be up & running with all the ports open.
The fact that the container is listening to the desired port does not mean that the container is fully ready.
That is why I had to add some Sleep code here and there within my tests to be 100% sure the apps where running.

```java
import static com.jayway.restassured.RestAssured.given;
import static org.hamcrest.CoreMatchers.containsString;

import org.hamcrest.Matchers;
import org.junit.BeforeClass;
import org.junit.ClassRule;
import org.junit.Test;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import com.mycompany.integration.mule.scenario.DockerMFXUpdatesAndDeletesScenario;
import com.mycompany.integration.rabbitmq.RabbitMqConnector;
import com.mycompany.integration.utils.SleepUtils;
import com.jayway.restassured.response.ValidatableResponse;
import com.palantir.docker.compose.DockerComposeRule;

/**
 * Mule Integration test Mule, ORIGIN, DESTINY, RabbitMq
 *
 */
public class MuleIntegrationTest {

    private String DESTINY_URL = "http://DESTINY:8090/app/rest/data/get?id=";

    private static RabbitMqConnector rabbitmq;

    private ExpectResponseDefinitions expectations = new ExpectResponseDefinitions();

    @ClassRule
    public static DockerComposeRule docker = DockerScenario.getDockerScenarioMuleTesting();

    @BeforeClass
    public static void initRabbitMQ() {
        SleepUtils.sleepSeconds(50);
        rabbitmq = new RabbitMqConnector();
    }

    @Test
    public void shouldSendActionToDestinyWhenDeleteMessageHRReceived() {
        whenDeleteHRMessageIsReceived();
        SleepUtils.sleepSeconds(5);
        thenDeleteHRActionWasSentToPalm();
    }

    private void whenDeleteHRMessageIsReceived() {
        rabbitmq.sendDeleteMessage("1234");
    }

    private void thenDeleteHRActionWasSentToPalm() {
        checkVersionActionMatchesForMediaId( expectations.expectedHRDelete("1234"));
    }

    private void checkVersionActionMatchesForMediaId(String expectedVersionAction) {
        ValidatableResponse response = given().when().get(urlCheckId("1234")).then();
        response.body("value", containsString(expectedVersionAction)).and()
                .body("id", Matchers.equalTo("1234")).statusCode(200);
    }

    private String urlCheckId(String mediaId) {
        return DESTINY_URL + mediaId;
    }
}
```

The important bit is around the ClassRule that is invoked before the test are launched spinning up all the containers defined by the scenario.

Ok, so now that we have our Junit test ready, the only remaining bit is our docker-compose.yml file where we define our services

<script src="https://gist.github.com/mfarache/5bf37a907b4a2bbadd041049128de904.js"></script>

+ DESTINY is the application I created with Spring Boot to record the requests
+ WIREMOCK-ENRICHER is the mocks responsible of providing metadata to MuleESB
+ MULE is our ESB layer, subject of integration black box testing
+ RABBITMQ is our Broker which I setup specifically with a couple of queues in my Dockerfile

Everything is run within a "integration-tier" network created to run test in isolation.

You can go to your IDE and run the unit test. Boom!

See output

Considerations about adding a step to build the images.

# Key takeaways

+ Use REST assured for easy REST testing of endpoints
+ Use palantir  Junit rule to launch a docker compose file with the definition of your scenario.
+ Use Docker for everything!

# Gotchas

There were no issues running the test under Mac Os or Linux.. however there are still uses with Windows, mainly
because by some reason Palantir uses a command "docker compose ps -q <servicename>" to detect if container is up and running. And the command fails.

# Useful links

+ [Arquillian Cube][1]
+ [Palantir Junit rule (github)][2]

[1]: http://arquillian.org/arquillian-cube/
[2]: https://github.com/palantir/docker-compose-rule
