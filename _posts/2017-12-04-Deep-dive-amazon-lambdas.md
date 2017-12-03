---
layout: post
title: Deep dive on amazon lambdas
tags: [ AWS, AWS Lambdas]
---

The following post will attempt to go through best practices, networking and things to bear in
mind while developing with AWS lambdas

# Best practices

## 1. Chose the right memory

We need to remember that CPU allocation is directly proportional to memory allocation.
What does it mean? If you double size of your memory you will get double of CPU assignment, which will lead to double cost in your monthly bill.

## 2. Use environment variables

Pretty obviuos as you should never have hardcoded values. AWS Lambda console allows to configure variables that can be easily accesed using
Java System.getEnv("VARIABLE")

## 3. Startup time

AWS provisions a  runtime container to execute your lambda. This process, referred to as a cold start, will increase your execution time considerably.
Cold start in AWS could vary between 3-5secs

Bear in mind that lambda code lifecycle goes through different stages:
 + initialisation : Executed only once in warm up time
 + invocation     : Executed every time your lambda is called

So if you have to deal with resource allocation, name resolution or targeting a database it would be advisable to do that on initialisation phase so the following invocation calls do not waste time.  Lambdas are meant to be stateless but when cost may be an issue it may be wiser to perform
your resource allocation during initialisation phase.

Let´s do some maths... initially we will rule out provisioning container time from the equation..

Imagine that initialisation of your resource takes 2sec and Let´s assume processing time of your lambdas last 0.25secs

If you do not follow this basic tip, 10 consecutive lambda invocations will last 22.5 secs
Following the tip will last 4.5secs.

Lambdas lifetime stay live during a period of time, so if you want to avoid the container running your lambda to be shutdown, you could do "warm up" calls to keep it alive, even if that means spending some tiny amount of compute time.  

Now let's consider that container provisioning time takes 5sec.
Assume that container running your lambda lifetime is 5 minutes (AWS does not say anything specific about that).
Once 5 minutes expire, the container will shut down and next invocation will spend considerable time till a new container is targeted to
execute your lambda. On top of that you will need 2 seconds to allocate your resources.

In one day you will spending (5+2)x12x5x24 secs of compute time. Nearly 3 hours wasted!
Assuming your daily cost is 8$, doing warm up you could save nearly 1$ just by providing a warm up mode of your lambda handler.

Your lambda handler could have a "dryRun" mode where no logic is executed.
Using CloudWatch you could trigger an event using dryRun = true and that way your bill will be reduced 12.5%
If we scale things up for proper production services where lambda invocations can be on the thousands scale per hours you can see the benefits.

## 4. Logging

Integration between CloudTrail and Lambdas is straight forward. It works like a charm in development mode and when you have just a few lambdas.
We know that running in production, we may end up with tens/hundreds of lambdas.. In that case you may be interested adding a hook with Elastic Search, Kibana, Logstash stack.

Luckily enough someone did that for us ;)
https://github.com/lukewaite/logstash-input-cloudwatch-logs

You specify a Log Group to stream from and this input plugin will find and consume all Log Streams within. The plugin will poll and look for streams with new records, and consume only the newest records.

## 5. Split business logic from your  lambda handler

One of the main drawbacks with AWS lambdas is that you cannot know whether or not your code will work till is deployed into the cloud.
In order to alleviate that a few tips on how to improve that restriction. First one is through unit testing your core business logic.
So try to keep your main lambda code as small as possible, so the handler lambda function delegates responsabilities to a class which is
decoupled from the lambda event.

Let´s start with an example so we can see some weaknesses of the alternative approach where everything is within the lambda.
The lambda function should connect with an existing DynamoDB and create a user based on the provided data.

```java
public class AwsCreateHandler  implements RequestHandler<CreateUserRequest, CreateUserResponse> {

    public CreateUserResponse handleRequest(CreateUserRequest req, Context context) {
        UserData user = new UserData();
        user.setName = req.getName();

        //Connect to database, and create the user
        ...

        CreateUserResponse response = new CreateUserResponse();
        response.setName = newUser.getName();
        response.setId = newUser.getId();
        return response;
    }
}
```

A possible naive approach for testing would be

```java
public class AwsCreateHandlerTest {

    private AwsCreateHandler handler;
    private TestContext testContext;

    @Before
    public void setUp() throws Exception {
        subject = new AwsCreateHandler();  
        testContext = new Context() {
            // implement all methods of this interface and setup your test context.
            // For instance, the function name:
            @Override
            public String getFunctionName() {
                return "AwsCreateLambda";
            }
        }
    }

    @Test
    public void should_handle_request() {
        CreateUserRequest req =  new CreateUserRequest("John")
        CreateUserResponse output = subject.handleRequest(input, testContext);
        assertNotNull(output);
        assertEquals(req.getUserName(), output.getUserName());
        assertNotNull(output.getId());
    }
}

```
Obviusly this will never work because our lambda function tries to create a user using a database and our unit test is not aware of that.

First thought would be create a service and its implementation.
Using DI we could mock the service itself on our unit test. After some googleing apparently we can get the benefits of Spring IOC as its show in this example
https://github.com/cagataygurturk/aws-lambda-java-boilerplate

With a quick refactoring using services our code will be way easier to test.

Our service interface definition

```java
package com.example.lambdatesting

public interface UserService {
    public User createUser(UserData data);
}
```

Our service implementation

```java
package com.example.lambdatesting
import org.springframework.stereotype.Component;

@Component
public class UserServiceImpl {
    public User createUser(UserData data) {
      //Connect to database, and create the user
      ...
    }
}
```

A helper class responsible of transforming lambda request/response model into our persistence object

```
package com.example.lambdatesting
import org.springframework.stereotype.Component;

@Component
public class ConverterUtils {
  public CreateUserResponse toResponse(User user) {
    CreateUserResponse response = new CreateUserResponse();
    response.setName = newUser.getName();
    response.setId = newUser.getId();
    return response;
  }

  public UserData fromRequest(CreateUserRequest req) {
    UserData user = new UserData();
    user.setName = req.getName();
    return user;
  }
}
```

A spring configuration class to provide the context we need for autowiring
```java
package com.example.lambdatesting

import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;


@Configuration
@ComponentScan("com.example.lambdatesting")
public class SpringConfig {

}
```

Our lambda code now is way cleaner and easy to test.
```java
package com.example.lambdatesting

public class AwsCreateHandler  implements RequestHandler<CreateUserRequest, CreateUserResponse> {

    //Assume we have a implementation based on environment variables of your DynamoDB via environment variables
    @Autowire
    public UserService userService;

    @Autowire
    public ConverterUtils converter;

    public CreateUserResponse handleRequest(CreateUserRequest req, Context context) {
        return converter.toResponse(userService.createUser(converter.fromRequest(req)));
    }
}
```

The approach allows 100% unit testing of the ConverterUtils class and our service implementation
On top of that now we can test our lambda properly using usual mocking practices.

```java
public class AwsCreateHandlerTest {

    private AwsCreateHandler handler;
    private TestContext testContext;

    @Mock
    User service;

    @Before
    public void setUp() throws Exception {
        MockitoAnnotations.initMocks(this)
        subject = new AwsCreateHandler();  
        testContext = new Context() {
            // implement all methods of this interface and setup your test context.
            // For instance, the function name:
            @Override
            public String getFunctionName() {
                return "AwsCreateLambda";
            }
        }
    }

    @Test
    public void should_handle_request() {
        CreateUserRequest req =  new CreateUserRequest("John")
        User user = new User("1", "John");
        when(service.createUser(any(User.class))).thenReturn(user);
        CreateUserResponse output = subject.handleRequest(input, testContext);
        assertNotNull(output);
        assertEquals(req.getUserName(), output.getUserName());
        assertEquals("1", output.getId());
    }
}

```

Now that we reached this point, a warning call about dependencies. The previous example was just conceptual.
If you require a complex framework to run your lambdas (Spring / Hybernate) you may find issue when AWS loads your lambdas
due to size limitation.  Lambda function deployment package size (compressed .zip/.jar file) cannot be more than 50MB

## 6. Test Your Serverless Applications Locally Using SAM Local

SAM Local is an AWS CLI tool that provides an environment for you to develop, test, and analyze your serverless applications locally before uploading them to the Lambda runtime.

It´s based on SAM aplication model

https://github.com/awslabs/serverless-application-model

You define your SAM templates that may imply definition of lambda functions, API gateways of other resources.
SAM local incorporates a http server environment where you can test your API defined in your template.
You can also generate mock events that can be used as input for the invocations of your lambdas.

I suggest reading http://docs.aws.amazon.com/lambda/latest/dg/test-sam-local.html#sam-cli-simple-app to get better understanding on the benefits of the approach
