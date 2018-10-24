---
layout: post
title: Using grpc as an alternative to REST for comms among microservices
tags: [ grpc, java, microservices, rest ]
---

Using gRpc as an alternative to REST for inter service communication among microservices.

Sometimes we embrace a technology without really thinking the drawbacks of using it.
Or we do not research if there is something better. Sometimes you stumble with it by chance
and suddenly you realize that better alternatives exist.

# What is wrong with REST



REST was designed to provide statufulness operations over resources using HTTP verbs.
Period. 

However there are several scenarios where really REST does not fit very well.
First that springs to mind is when we have to implement an operation that necessarily does not relate with a single resource. Who has not created sometimes an endpoint to trigger  "something" which
implies several operations? Or to trigger simply a process? Or to expose the resultset
of a specific business query? 

If we use REST for inter-service communication you also must admit that precious time is wasted spent marshalling / unmarshalling our POJO objects. One of the main drivers on why REST was so cool was simply because using Json or XML we were able to understand what's going on over the wire. But is that the right thing when machines are supposed to understand binary protocols?

There are also other drawback stemed from the fact that we are using REST over http and some limitation inherent to the protocol itself. At least with Http1.1 there are few things that do not help in terms of efficiency. 

# Welcome gRPC

First reaction probably would be to step back jumping...  as you may associate it with old Remote Procedure Calls in the past. Nightmarish times comes to my mind those times where in the benefit of interoperability we used CORBA, or RMI, etc. Nor the tools were ready for
the job and luckily enough EJBs using RMI are something hopefully from the past.

Let's focus our context on microservices intercomunication and let's figure whether or not gRPC may be a good option

Some of the obvious benefits are:

## Efficiency

GRPC uses Protobuffer for binary serialization of messages, which gives a lot more efficiency when compared to textual JSON used by REST.
The efficiency is also better from CPU perspective. Now that we are moving towards a serverless world ruled by virtual instances, lambdas and
pay-as-you-go models,  CPU efficiency would reduce significantly your monthly bills with your cloud provider.
As gRPC is pure Binary protocol, there is no need of encode/decode json/xml payload in both ends of the pipe.

## Faster

grpc is built on top of HTTP2 so for example we can use compression of headers, two-way comunication and interleaved multiplexing over the same channel. Instead of the tipical latency originated by the request-response TCP handshake , with http2 we can get much better throughput on our channel. The guys who developed protocols-buffer claim that speeds improvement are in the magnitued of 20-100 times faster when compared with xml/json transfer payloads. 

## Strongly typed

I really do not get well with languages like perl,ruby,python that are not strongly typed. Or duck typing family. I think those are a devilish recipe created to make our lifes harder.  

I reckon I deviated from the topic of the blog, but I think you get the analogy. Generally speaking REST deals both json or xml payload. 
The client consuming the response needs to be changed if the server removes a field

However returning more fields in the payload is not forbidden unless your client does a strict JSON validation, which most of the times is not the case. 
With complex JSON payloads with nested and subnested elements is not straightforward to create our POJO classes and very often we need to create then manually.

The good thing about grpc is that a definition of a strongly type payload message and the service call defintions is everything you need to guarante interoperability.

## Streaming

gRPC takes full advantage of the capabilities of HTTP/2  mechanism. 
It supports  streaming at request level (multiple requests sent to the server), at response level (server replies a stream of response), or both request and responses streamed over.

Ok, so if everything is awesome why do we still use REST ? 
Well ..., remember that the context of this post is about microservices.

There is a huge ecosystem out there of web applications delivering xml/json to the browsers and the 
frontend clients are built using Angular, React, Vue, etc using traditional REST clients.
Support of http2 in the browser is not something that Javascript are ready to handle well yet. Also I can imagine debugging tons of endpoint response without the existance of some plugin able to interpret the binary response would make life of web developers a real pain. 

# The gaming server 

Best way to learn is with a practical example. We will build a very simple 
gaming server to cope 3 basic features. 

The server will be able to track scores and users for a subset of games.
+ User may register in the server for a specific game
+ User may submit a score of the game
+ User may request which users are playing the same game
+ User may subscribe to be informed about the ranking hall of fame on a game
+ Server will inform everyone about the hall of fame for a game

 ![_config.yml]({{ site.baseurl }}/images/GRPC.jpeg)

This will give the chance to go throuh the stages end to end:

+ Service definition understanding the messages format
+ Generation of service stubs based on service definition
+ Implement a Sever
+ Implement different clients types (sync, async, futures)



## Service definition 

We neeed to define the format of our messages using protocol buffers.

They are defined in a .proto file

It's a pseudo-language to define structures . Strontgly typed. Yes!

It allows definition hierarchical nested types, use of enum types,
importing other .proto files, defining mandatory / optional values, etc.

One small limitation is that is not possible to define services one-way only
so for that purpose we create a message GamingServerResponse type to hold the server response in that case with an enum indicating whether response was fine or failed.

```c

syntax = "proto3";

package myexamples;

option java_multiple_files = true;
option java_package = "com.gaming.grpc";
option java_outer_classname = "GamingServer";

message Game {
    string name =1;
}

message User {
    string username =1;
    string email=2;
    Game game=3;
}

message Score {
    string username =1;
    int32  points=2;
    Game game=3;
}


message TopNHallOfFameRequest {
    Game game=1;
    int32 howMan=2;
}

message HallOfFame {
    repeated Score users =1;
}

message GamingServerResponse {
  enum StatusType {
        OK = 0;
        ERR = 1;
  }
  StatusType status =1;
}

service GamingServer {

    rpc AddUser(User) returns (GamingServerResponse) {}

    rpc AddScore(stream Score) returns (GamingServerResponse) {}

    rpc GetUsers(Game) returns (stream User) {}

    rpc GetHallOfFame(TopNHallOfFameRequest) returns (stream HallOfFame) {}

}
```

We have 2 options to generate  data access classes and stubs to enable generation of client and servers.

+ Run the protocol buffer compiler for JAVA ( or your language of choice)
on our .proto file  

+ Use maven plugin, which is my preferred approach as can be easily automated with our build process.

```bash

$ cd $HOME Documents/workspace/grpc-gaming-server/
$ mvn clean install
#Generation of source code
$ tree ./target/generated-sources
└── protobuf
    ├── grpc-java
    │   └── com
    │       └── example
    │           └── gaming
    │               └── GamingServerGrpc.java
    └── java
        └── com
            └── example
                └── gaming
                    ├── Game.java
                    ├── GameOrBuilder.java
                    ├── GameServer.java
                    ├── GamingServerResponse.java
                    ├── GamingServerResponseOrBuilder.java
                    ├── HallOfFame.java
                    ├── HallOfFameOrBuilder.java
                    ├── Score.java
                    ├── ScoreOrBuilder.java
                    ├── TopNHallOfFameRequest.java
                    ├── TopNHallOfFameRequestOrBuilder.java
                    ├── User.java
                    └── UserOrBuilder.java

```
We can observe a few things from the generated code

+ There is a Java class with the same name of our proto file (GameServer) 
+ There is a java class per message type 
+ Defintion of our messages will be done using inmutable objects generatadated via Builder pattern 
+ We have a class GamingServerGrpc which contains base classes to extend from 
when we start creating our server.


## Building the Gaming server

A few things to mention about the grpc Server is that is Reactive by default.
The decision about about behaviour is done by the client at the moment of issuing the service call. 

The server code is straight forward, we just register a handler to GameService which
really implements all the calls defined by our proto messages.
As you do not want your server to end up immediately add awaiTermination so execution does not ends.

```java
public class GamingServer {
	
	private static final Logger LOG = Logger.getLogger(GamingServer.class.getName());

	private Server server;

	public static void main(String[] args) throws IOException, InterruptedException {
		final GamingServer server = new GamingServer();
		server.start();
		server.blockUntilShutdown();
	}

	private void start() throws IOException {
		ServerBuilder serverBuilder = ServerBuilder.forPort(GameServerSettings.PORT).addService(new GameService());
		server = serverBuilder.build().start();
		LOG.info("Server started, listening on " + GameServerSettings.PORT);

		Runtime.getRuntime().addShutdownHook(new Thread() {
			@Override
			public void run() {GamingServer.this.stop();

			}
		});
	}

	private void stop() {
		if (server != null) {
			System.err.println("*** shutting down gRPC server since JVM is shutting down");
			server.shutdown();
			System.err.println("*** server shut down");
		}
	}

	private void blockUntilShutdown() throws InterruptedException {
		if (server != null) {
			server.awaitTermination();
		}
	}

}

```

Our GameService overrides the stub so we can implement our custom logic.
The server tracks information about the users and scores in memory for simplicity.

We will see a couple of rpc definitions and its implementations as methods

AddUser was the simpler one. We can notice how the response is translated into a StreamObserver.

```c
  rpc AddUser(User) returns (GamingServerResponse) {}
```

```java
	@Override
	public void addUser(User request, StreamObserver<GamingServerResponse> responseObserver) {
		LOG.info("--- addUser request");
		Game game = request.getGame();
		if (usersPerGame.containsKey(game)) {
			usersPerGame.get(game).add(request);
			responseObserver.onNext(MessageBuilder.okResponse());
		} else {
			responseObserver.onNext(MessageBuilder.errorResponse());
			responseObserver.onError(new IllegalArgumentException("Inexisting Game:" + game.getName() ));
		}

		responseObserver.onCompleted();
	}
```



Users can subscribe to HallOfFame updates using this method.
Here we do notreturning the Hall Of Fame response inmmediately. 
Instead we add the observer to a list hallOfFameSubscribers. 
Whenever we receive new Scores, we can iterate over the subscribers so we can inform to our clients.

Also notice we are not calling "responseObserver.onCompleted()" otherwise stream will be closed


```c
  rpc GetHallOfFame(TopNHallOfFameRequest) returns (stream HallOfFame) {}
```

```java
	@Override
	public void getHallOfFame(TopNHallOfFameRequest request, StreamObserver<HallOfFame> responseObserver) {
		LOG.info("--- getHallOfFame request");
		Game game = request.getGame();

		if (scorePerGame.containsKey(game)) {
			hallOfFameSubscribers.add(responseObserver);
		} else {
			responseObserver.onError(new IllegalArgumentException("Inexisting Game:" + game.getName()));
		}
		// We do not want to close the stream as we are interested on sending updates to
		// all our consumers when they submit scores.
		// responseObserver.onCompleted();
	}
```

The last one involves client streaming, so is a bit more difficult to understand at a glance

```c
  rpc AddScore(stream Score) returns (GamingServerResponse) {}
```

When we use streaming, things are slightly more complex and probably anyone can struggle to understand the meaning of the parameters and the response if you have your head "furnished" into a request-response way

The input parameter to our method is what the server needs to be send to the client.
You can see it as a listener that handles the messages coming from the server.

The response is really what what the client sends to the server. Or explained in a different way it returns an object that is getting data from the client to the server.

```java
	@Override
	public StreamObserver<Score> addScore(StreamObserver<GamingServerResponse> responseObserver) {

		LOG.info("--- AddScore request");

		return new StreamObserver<Score>() {
			@Override
			public void onCompleted() {
				
			}

			@Override
			public void onError(Throwable arg0) {
				LOG.info("ERROR" + arg0);
			}

			@Override
			public void onNext(Score score) {
				LOG.info(String.format("Score: %s %s", score.getUsername(), score.getPoints()));
				Game game = score.getGame();
				if (scorePerGame.containsKey(game)) {
					scorePerGame.get(game).add(score);
					hallOfFameSubscribers.forEach(subscriber -> {
						LOG.info("About to notify: " + subscriber.toString());
						notifyConsumer(subscriber, calculateTopScores(game, GameServerSettings.HALL_OF_FAME_RANK));
					});
					responseObserver.onNext(MessageBuilder.okResponse());

				} else {
					responseObserver.onNext(MessageBuilder.errorResponse());
					responseObserver.onError(new IllegalArgumentException("Inexisting Game:" + game.getName()));
				}
				responseObserver.onCompleted();
			}
		};
}
```

## Building the client

The client abstracts the complexities of the HTTP2 transport using concepts like Channels
to represent the connection to the grpc server.

A channel can manage loadbalancing and decide LB RoundRobin strategy.
To do that you need a NameResolver and for example we could hook it with Eureka to get the service using the logical NameResolver, but that is out of scope for this blog.


If we were using TLS we would need to specify that when building our channel.
However as I was not able to start up the server with TLS enabled we need to create our client to use plain text.

```java
Channel channel = ManagedChannelBuilder.forAddress(GameServerSettings.HOST, GameServerSettings.PORT).usePlaintext().build();
```

I have created 3 flavours of the client stub to see how they work 

+ AsyncGameServiceClient - Fully asyncronous - that listen to stream observer call backs
+ FutureGameServiceClient - Return futures so lazily we can evaluate its result
+ BlockingGameServiceClient  - block the call till the stream returns

Regardless of the type of client all of them will behave the same

+ Try to register a user against a random game (via java args).
+ Try to get the top N users for that game.

What about scores? Well no matter that we try getting that using blocking or futures.
In fact the stub method is not generated. You would be wondering why?
The answer is obvious once you think it through.  

Our rpc call defintion was:

```c
rpc AddScore(stream Score) returns (GamingServerResponse) {}
```

So really how can we stream in a blocking (request-response way?)

Therefore only AsyncGameServiceClient can provide the feature to add Scores.

Let's wrap up this post seeing the client implementation for each of them.

## FutureGameServiceClient

Future style do not raise any surprise.

```java
public class FutureGameServiceClient {

	private static final Logger LOG = Logger.getLogger(FutureGameServiceClient.class.getName());

	public static void main(String[] args) throws InterruptedException, ExecutionException {
		
		Channel channel = ManagedChannelBuilder.forAddress(GameServerSettings.HOST, GameServerSettings.PORT).usePlaintext().build();
		Game game = MessageBuilder.aGame();
		User user = MessageBuilder.aUser(args[0], args[1], game);

		GamingServerFutureStub futureClient = futureClient(channel);
		ListenableFuture<GamingServerResponse> futureListenerAddUser = futureClient.addUser(user);
		GamingServerResponse response = futureListenerAddUser.get();
		LOG.info("Status:" + response.getStatus());

		ListenableFuture<HallOfFame> futureListenerGetHallOfFame = futureClient
				.getHallOfFame(MessageBuilder.aTopRequest(game, 5));
		HallOfFame hallOfFame = futureListenerGetHallOfFame.get();
		LOG.info("HallOfFame:" + hallOfFame.getUsersCount());

	}

	private static GamingServerFutureStub futureClient(Channel channel) {
		return GamingServerGrpc.newFutureStub(channel);
	}

}
```

## BlockingGameServiceClient

Blocking style is also trivial

```java
public class BlockingGameServiceClient {

	private static final Logger LOG = Logger.getLogger(BlockingGameServiceClient.class.getName());

	public static void main(String[] args) throws InterruptedException, ExecutionException {
		
		Channel channel = ManagedChannelBuilder.forAddress(GameServerSettings.HOST, GameServerSettings.PORT).usePlaintext().build();
		Game game = MessageBuilder.aGame();
		User user = MessageBuilder.aUser(args[0], args[1], game);
		

		GamingServerBlockingStub blocking = blockingClient(channel);
		
		GamingServerResponse addUserResponse = blocking.addUser(user);
		LOG.info("Status:" + addUserResponse.getStatus());
		
		blocking.getUsers(game).forEachRemaining(u->LOG.info("user:" + u.getUsername()));
		
		HallOfFame hallOfFame = blocking.getHallOfFame(MessageBuilder.aTopRequest(game, 5));
		LOG.info("HallOfFame:" + hallOfFame.getUsersCount());

	}

	private static GamingServerBlockingStub blockingClient(Channel channel) {
		return GamingServerGrpc.newBlockingStub(channel);
	}

}
```

## AsyncGameServiceClient

We will focus in the async one eventually as is the most interesting, so we will see how it works spinning server and a couple of clients.

The client will always use the same game, so multiple client instances can contribute to the same Hall Of Fame (remember the server supports multiple games).

To create a more realistic example the client can be parameterized via java args so we can simulate different users accessing our server and submitting random scores.

```java
public class AsyncGameServiceClient {

	private static final Logger LOG = Logger.getLogger(AsyncGameServiceClient.class.getName());
	private static GamingServerStub async;

	public static void main(String[] args) throws InterruptedException, ExecutionException {

		Channel channel = ManagedChannelBuilder.forAddress(GameServerSettings.HOST, GameServerSettings.PORT)
				.usePlaintext().maxInboundMessageSize(16 * 1024 * 1024).build();
		Game game = MessageBuilder.aGame(GameServerSettings.validGameNames.get(0));
		User user = MessageBuilder.aUser(args[0], args[1], game);

		async = asyncClient(channel);

		async.addUser(user, new StreamObserver<GamingServerResponse>() {
			@Override
			public void onCompleted() {

			}

			@Override
			public void onError(Throwable arg0) {
				LOG.log(Level.WARNING, "addUser:" + arg0.getMessage(), arg0);
			}

			@Override
			public void onNext(GamingServerResponse arg0) {
				LOG.info("addUser - Status:" + arg0.getStatus());

			}
		});

		Thread.sleep(1000);
		getUsers(game);

		Thread.sleep(1000);
		hallOfFame(GameServerSettings.HALL_OF_FAME_RANK, game);

		int N = new Random().nextInt(5);
		int[] values = { N * 1000, N * 2000, N * 3000, N * 4000 };
		IntStream.of(values).forEach(score -> {
			try {
				Thread.sleep(1000);
				LOG.info("**** Adding score: " + score);
				addScore(user.getUsername(), score, game);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		});

		Thread.sleep(100000);

	}

	private static void addScore(String user, int points, Game game) {
		StreamObserver<Score> scoreObserver = async.addScore(new StreamObserver<GamingServerResponse>() {
			@Override
			public void onCompleted() {};

			@Override
			public void onError(Throwable arg0) {
				LOG.log(Level.WARNING, "addScore:" + arg0.getMessage(), arg0);
			}

			@Override
			public void onNext(GamingServerResponse response) {
				LOG.info("addScore - Status:" + response.getStatus());
			}
		});
		scoreObserver.onNext(MessageBuilder.aScore(points, user, game));
	}

	private static void hallOfFame(int n, Game game) {
		async.getHallOfFame(MessageBuilder.aTopRequest(game, n), new StreamObserver<HallOfFame>() {
			@Override
			public void onCompleted() {};

			@Override
			public void onError(Throwable arg0) {
				LOG.log(Level.WARNING, "getHallOfFame:" + arg0.getMessage(), arg0);
			}

			@Override
			public void onNext(HallOfFame hallOfFame) {
				LOG.info("******************* Hall of fame Rank **********************");
				int[] index = { 1 };
				hallOfFame.getUsersList().forEach(u -> {
					LOG.info("******* " + (index[0]++) + " : " + u.getUsername() + " SCORE:" + u.getPoints());
				});
				LOG.info("******************* Hall of fame Rank **********************");
			}
		});
	}

	private static void getUsers(Game game) {
		async.getUsers(game, new StreamObserver<User>() {
			@Override
			public void onCompleted() {};

			@Override
			public void onError(Throwable arg0) {
				LOG.log(Level.WARNING, "getUsers:" + arg0.getMessage(), arg0);
			}

			@Override
			public void onNext(User user) {
				LOG.info("Existing User:" + user.getUsername());
			}
		});
	}

	private static GamingServerStub asyncClient(Channel channel) {
		return GamingServerGrpc.newStub(channel);
	}

}
```

We can trigger our server 

```bash
mvn exec:java -Dexec.mainClass="com.grpc.gamingserver.server.GamingServer" 

```
Open a new term session and let's start our async client. We will refer to this as Session#1

```bash
mvn exec:java -Dexec.mainClass="com.grpc.gamingserver.AsyncGameServiceClient" -Dexec.args="user1 user1@gmail.com" 
````

We can see that every time the user submits a score, he also receive a HallOfFame using the open stream we have. As we have only one user connected to our server he is the only one in the HallOfFame based on the scores submitted by user1. 

```bash
NFO: ******************* Hall of fame Rank **********************
Oct 24, 2018 12:48:40 PM com.grpc.gamingserver.AsyncGameServiceClient$3 lambda$0
INFO: ******* 1 : user1 SCORE:12000
Oct 24, 2018 12:48:40 PM com.grpc.gamingserver.AsyncGameServiceClient$3 lambda$0
INFO: ******* 2 : user1 SCORE:9000
Oct 24, 2018 12:48:40 PM com.grpc.gamingserver.AsyncGameServiceClient$3 lambda$0
INFO: ******* 3 : user1 SCORE:6000
Oct 24, 2018 12:48:40 PM com.grpc.gamingserver.AsyncGameServiceClient$3 lambda$0
INFO: ******* 4 : user1 SCORE:3000
Oct 24, 2018 12:48:40 PM com.grpc.gamingserver.AsyncGameServiceClient$3 onNext
INFO: ******************* Hall of fame Rank **********************
```
We can spice things a little by adding as many clients as you want, each of them with a different username. To see results clearly is better to execute each command in a different terminal session. We will just create a Session#2 

```bash
mvn exec:java -Dexec.mainClass="com.grpc.gamingserver.AsyncGameServiceClient" -Dexec.args="user2 user2@gmail.com" 
````

If we switch back to Session#1 we can see now that the Server is streaming results into the Hall Of Fame considering user1 and user2:

```bash
INFO: ******************* Hall of fame Rank **********************
Oct 24, 2018 12:50:14 PM com.grpc.gamingserver.AsyncGameServiceClient$3 lambda$0
INFO: ******* 1 : user1 SCORE:12000
Oct 24, 2018 12:50:14 PM com.grpc.gamingserver.AsyncGameServiceClient$3 lambda$0
INFO: ******* 2 : user2 SCORE:12000
Oct 24, 2018 12:50:14 PM com.grpc.gamingserver.AsyncGameServiceClient$3 lambda$0
INFO: ******* 3 : user1 SCORE:9000
Oct 24, 2018 12:50:14 PM com.grpc.gamingserver.AsyncGameServiceClient$3 lambda$0
INFO: ******* 4 : user2 SCORE:9000
Oct 24, 2018 12:50:14 PM com.grpc.gamingserver.AsyncGameServiceClient$3 lambda$0
INFO: ******* 5 : user1 SCORE:6000
Oct 24, 2018 12:50:14 PM com.grpc.gamingserver.AsyncGameServiceClient$3 onNext
INFO: ******************* Hall of fame Rank **********************
```
I also explored unit testing (See and added an example in the source code if you are curious. However seems the stub gets hang while doing the call, and could not figure out why.

```java
@RunWith(JUnit4.class)
public class GamingServerTest {

	private static final String USER = "user";
	private static final String USER_GMAIL_COM = "user@gmail.com";

	@Mock
	GameService gameService;

	@Rule
	public GrpcCleanupRule grpcCleanup = new GrpcCleanupRule();

	private String serverName;
	private InProcessServerBuilder serverBuilder;
	private InProcessChannelBuilder channelBuilder;

	@Before
	public void init() {
		MockitoAnnotations.initMocks(this);
		serverName = InProcessServerBuilder.generateName();
		serverBuilder = InProcessServerBuilder.forName(serverName).directExecutor();
		channelBuilder = InProcessChannelBuilder.forName(serverName).directExecutor();
		registerServiceAndStart();
	}

	@Test
	public void clientCallShouldTriigerAddUserInTheServer() {
		GamingServerBlockingStub blockingClient = GamingServerGrpc.newBlockingStub(getChannel());
		GamingServerResponse response = blockingClient.addUser(aUser(aGame()));
		assertEquals(StatusType.OK, response.getStatus());

		Mockito.verify(gameService).addUser(Mockito.argThat(new ArgumentMatcher<User>() {
			@Override
			public boolean matches(Object argument) {
				User user = (User) argument;
				return user.getEmail().equals(USER_GMAIL_COM) && user.getUsername().equals(USER);
			}
		}), Mockito.argThat(new ArgumentMatcher<StreamObserver<GamingServerResponse>>() {
			@Override
			public boolean matches(Object argument) {
				return true;
			}
		}));
	}

	private ManagedChannel getChannel() {
		return grpcCleanup.register(channelBuilder.build());
	}

	private void registerServiceAndStart() {
		try {
			grpcCleanup.register(serverBuilder.addService(gameService).build().start());
		} catch (IOException e) {
			fail();
		}
	}

	private User aUser(Game game) {
		return User.newBuilder().setUsername(USER).setEmail(USER_GMAIL_COM).setGame(game).build();
	}

	private Game aGame() {
		Game game = Game.newBuilder().setName("Game").build();
		return game;
	}
}
```

There are more interesting topics worthy exploring like performance, mainly how many connection can be kept open in case we were using a service that receives massive connections from multiple connections.

All the code is availabe here: [Code in github][4]

# Useful links

+ [Grpc getting started with java][1]
+ [Securing gRpc server with TLS][2]
+ [Load Balancing][3]
+ [Code in github][4]
+ [Clean Grpc resources on your unit tests][5]

[1]: https://grpc.io/docs/tutorials/basic/java.html
[2]: https://bbengfort.github.io/programmer/2017/03/03/secure-grpc.html
[3]: https://grpc.io/blog/loadbalancing
[4]: https://github.com/mfarache/grpc-gaming-server
[5]: https://grpc.io/blog/gracefully_clean_up_in_grpc_junit_tests













































