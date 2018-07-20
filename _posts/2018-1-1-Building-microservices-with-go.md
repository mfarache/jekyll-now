---
layout: post
title: Building microservices with Go, Gin and NATS
tags: [ Go, microservices, Gin, NATS ]
---

This post describes a simple approach to build microservices using Go language.
On the way we will apply  

# A video business case ( kind of fake one!)

Our domain will be around collecting video metadata and retrieval from different sources. Also we need to be able to track some stats around its viewing.

** IMPORTAN NOTE: ** Please note that this is a over simplification on the real use case where more services would be likely involved like license generation, user entitlement checking or georeference protection to name a few.

So we can assume that when a user tries want to watch a video, we need to figure out a few metadata that lives in different repositories:


+ Medatata related to the video itself like title, summary, genre
+ URL where the video is published so the web page can play it

At the same time we want to track events to indicate the number of times John Doe viewed any movie and the number of times a movie was viewed by any user.

# Breaking down the problem in small microservices

There are multiple articles about best practices when building microservices. I will stick to the basics:

 + do one thing and do it right, define a clear boundary
 + each microservice should own its own database
 + microservices should be loosely coupled.
 + share nothing
 + something that can be hacked and wiped out entirely in less than a day
 + services should be easily monitored, traceable and scalable

One that I consider important is that  should be reactive and event oriented. You will see that I did not follow strictly that approach as you will
see that some services depend on others aplying a request-response pattern between them.

We will build several microservices with the purpose to explore inter service communication patterns and typical issues we found while facing a ms architecture.

+ Asynchronous communication between Microservices using pub/sub queues  
+ Synchronous communication between Microservices using REST
+ A main edge service to aggregate results from downstream services
+ Each service will own its own database ( for simplicity we will simple in-memory maps)
+ Microservices health checks
+ Service discovery using Consul
+ Explore possibilities using  ist.io, linkerd and Kubernetes

Due to the extension of the subject I will split these blogs in several parts.We will start our solution with the simplest approach


# Golang to the rescue

I will use Go as implementation language as an excuse to learn a new language and see why all the hype about it.

So bear with me as I'm totally new, so probably some of the decision I take are wrong due to lack of experience in the language.

# Services definition

The following diagram shows a high level view of what we will build.

![_config.yml]({{ site.baseurl }}/images/GOMS-DIAGRAM.png)

We have 3 main downstream micro services which are listed below.

## Metadata service

Port: 8081
  + GET content/asset/:assetId
  + POST content/asset/:assetId

```go
package main

import (
    "flag"
    "fmt"

    "github.com/gin-gonic/gin"
)

type typeContent struct {
    title   string
    genre   string
    summary string
}

type keyContent struct {
    Key string
}

var (
    port    = flag.String("port", "8081", "Listeninng port")
    assetDB = make(map[keyContent]typeContent)
)

func init() {
    flag.Parse()
}

func createAssetDB() {
    assetDB[keyContent{"123"}] = typeContent{"It", "horror", "A very scary movie"}
    assetDB[keyContent{"456"}] = typeContent{"Terminator", "action", "Summary for terminator"}
    fmt.Printf("%+v\n", assetDB)
}

func createVideoContentRouter() *gin.Engine {

    createAssetDB()

    r := gin.Default()

    r.GET("content/asset/:assetId", handleGetContent)
    r.POST("content/asset/:assetId", handlePostContent)

    return r
}

func handleGetContent(c *gin.Context) {

	id := c.Params.ByName("assetId")

	value, ok := assetDB[keyContent{id}]
	if ok {
		c.JSON(200, gin.H{"id": id,
			"title":   value.title,
			"genre":   value.genre,
			"summary": value.summary})
	} else {
		c.JSON(404, gin.H{"id": id, "status": "Asset not found"})
	}
}

func handlePostContent(c *gin.Context) {

	var json typeContent
	id := c.Params.ByName("assetId")
	if c.Bind(&json) == nil {
		assetDB[keyContent{id}] = json
		c.JSON(200, gin.H{"status": "ok"})
	}

}

func main() {
	serverPort := fmt.Sprintf(":%s", *port)
	r := createVideoContentRouter()
	r.Run(serverPort)

}
```

## Reporting  service
Port: 8082
  + POST events/asset/:assetId/user/:userId
  + GET events/user/:userId"
  + GET events/asset/:assetId"

```go
package main

import (
	"flag"
	"fmt"

	"github.com/gin-gonic/gin"
)

type mapKey struct {
	Key string
}

var (
	port         = flag.String("port", "8082", "Listeninng port")
	userviewsDB  = make(map[mapKey]int)
	assetviewsDB = make(map[mapKey]int)
)

func init() {
	flag.Parse()
}

func setupRouter() *gin.Engine {

	r := gin.Default()

	r.POST("events/asset/:assetId/user/:userId", registerViewPerUserAndAsset)

	r.GET("events/user/:userId", viewsPerUser)

	r.GET("events/asset/:assetId", viewsPerAsset)

	return r
}

func registerViewPerUserAndAsset(c *gin.Context) {
	assetId := c.Params.ByName("assetId")
	assetCounter, ok := assetviewsDB[mapKey{assetId}]
	if ok {
		assetviewsDB[mapKey{assetId}] = assetCounter + 1
	} else {
		assetviewsDB[mapKey{assetId}] = 1
	}

	userId := c.Params.ByName("userId")
	userCounter, ok := userviewsDB[mapKey{userId}]
	if ok {
		userviewsDB[mapKey{userId}] = userCounter + 1
	} else {
		userviewsDB[mapKey{userId}] = 1
	}
	c.JSON(200, gin.H{"userviews": userviewsDB[mapKey{userId}],
		"assetviews": assetviewsDB[mapKey{assetId}]})
}

func viewsPerUser(c *gin.Context) {
	userId := c.Params.ByName("userId")

	value, ok := userviewsDB[mapKey{userId}]
	if ok {
		c.JSON(200, gin.H{"id": userId,
			"views": value})
	} else {
		c.JSON(404, gin.H{"id": userId, "status": "User not found"})
	}
}

func viewsPerAsset(c *gin.Context) {
	assetId := c.Params.ByName("assetId")
	value, ok := assetviewsDB[mapKey{assetId}]
	if ok {
		c.JSON(200, gin.H{"id": assetId,
			"views": value})
	} else {
		c.JSON(404, gin.H{"id": assetId, "status": "Asset not found"})
	}
}

func main() {
	serverPort := fmt.Sprintf(":%s", *port)
	r := setupRouter()
	r.Run(serverPort)
}


```

## CDN Url service
  Port: 8083
    + GET url/asset/:assetId
    + POST url/asset/:assetId

```go
package main

import (
	"flag"
	"fmt"

	"github.com/gin-gonic/gin"
)

type typeURL struct {
	url string
}

type keyURL struct {
	Key string
}

var (
	port  = flag.String("port", "8083", "CDN URL Listeninng port")
	urlDB = make(map[keyURL]typeURL)
)

func init() {
	flag.Parse()
}

func createDB() {
	urlDB[keyURL{"123"}] = typeURL{"path/to/asset123.mp4"}
	fmt.Printf("%+v\n", urlDB)
}

func createCDNUrlContentRouter() *gin.Engine {

	createDB()

	r := gin.Default()

	r.GET("url/asset/:assetId", handleGetURL)
	r.POST("url/asset/:assetId", handlePostURL)

	return r
}

func handleGetURL(c *gin.Context) {

	id := c.Params.ByName("assetId")

	value, ok := urlDB[keyURL{id}]
	if ok {
		c.JSON(200, gin.H{"id": id,
			"url": value.url})
	} else {
		c.JSON(404, gin.H{"id": id, "status": "Asset not found"})
	}
}

func handlePostURL(c *gin.Context) {

	id := c.Params.ByName("assetId")
	var json typeURL
	if c.Bind(&json) == nil {
		urlDB[keyURL{id}] = json
		c.JSON(200, gin.H{"status": "ok"})
	}
}

func main() {
	serverPort := fmt.Sprintf(":%s", *port)
	r := createCDNUrlContentRouter()
	r.Run(serverPort)

}

```

These services can be considered as backend services which will not be exposed directly to our enduser. They perform basic CRUD operations on isolated databases (in memory maps).
I provided POST endpoints just to allow adding data externally by an admin interface that could be easily scripted via bash script if required.

By default *cdnurl* and *metadata*  data is populated with default values for assets with 123 and 456.

We mentioned these services are not intended to be used by our frontend SPA application. So how can we get data from there?

Several options could be explored:
 + Using AWS API gateway to forward requests to those services ( for those familiar with AWS lambdas)
 + Using [Kong API Gateway][7] as API gateway. That would be great as we could take benefit on rate limiting features, logging or plugins available.
 + Using NGINX as reverse proxy to route our requests to the services.

Any of the previous options would work perfectly if there is no need to do something extra aside of the aggregation/orchestration.
However the SPA frontend would know too much about the underlying microservice structure, i.e more coupled.
Therefore I created an edge play service which aggregates content from downstream services ( *cdnurl* and *metadata* ) and emit events to record stat.

## Play service
 + POST events/asset/:assetId/user/:userId

The microservice uses a request-response pattern to communicate with *cdnurl* and *metadata*

As part of our POC I wanted to send asynchronous services, so I decided that our edge service will not speak directly with our *reporting* service.
Instead our play service will just emit events to be published in a "events" queue.

```go
package main

import (
	"client"
	"client/types"
	"flag"
	"fmt"
	"log"
	"queue"

	"github.com/gin-gonic/gin"
)

type PlayResponse struct {
	Genre      string `json:"genre"`
	ID         string `json:"id"`
	Summary    string `json:"summary"`
	Title      string `json:"title"`
	UserViews  int    `json:"userviews"`
	AssetViews int    `json:"assetviews"`
	Url        string `json:"url"`
}

var (
	port = flag.String("port", "8080", "Listeninng port")
)

func init() {
	flag.Parse()
}

func createPlayRouter() *gin.Engine {
	r := gin.Default()

	r.POST("play/asset/:assetId/user/:userId", handlePlayContent)
	return r
}

func handlePlayContent(c *gin.Context) {
	assetId := c.Params.ByName("assetId")
	userId := c.Params.ByName("userId")

	recordContent := client.GetAssetDetail(assetId)
	types.TraceContent(recordContent)

	recordUrl := client.GetAssetURL(assetId)
	types.TraceURL(recordUrl)

	var nc, err = queue.Connect("server")
	if err != nil {
		log.Fatal(err)
	} else {
		message := fmt.Sprintf("%s:%s", assetId, userId)
		queue.PublishMessage(nc, "events", message)

	}

	recordStats := client.GetStats(assetId, userId)
	types.TraceStats(recordStats)

	c.JSON(200, gin.H{"Id": recordContent.ID,
		"title":      recordContent.Title,
		"summary":    recordContent.Summary,
		"assetviews": recordStats.UserViews,
		"userviews":  recordStats.AssetViews,
		"url":        recordUrl.Url})
}



func main() {
	serverPort := fmt.Sprintf(":%s", *port)
	r := createPlayRouter()
	r.Run(serverPort)
}


```
We can see that the play service calls different downstream via the client package.
I added a client function per target. You can check how easy is making REST calls using the standard library and mapping the Json response to a struct type definition.

```go
package client

import (
	"client/types"
	"encoding/json"
	"fmt"
	"log"
	"net/http"
)

const HOST_PORT_CONTENT_SERVICE = "localhost:8081"

func GetAssetDetail(assetId string) types.Content {
	var record types.Content
	//Get content
	url := fmt.Sprintf("http://%s/content/asset/%s", HOST_PORT_CONTENT_SERVICE, assetId)
	// Build the request

	req, err := http.NewRequest("GET", url, nil)
	if err != nil {
		log.Fatal("Obtaining content failed: ", err)
		return record
	}

	client := &http.Client{}

	response, err := client.Do(req)
	if err != nil {
		log.Fatal("Do: ", err)
		return record
	}

	defer response.Body.Close()

	// Use json.Decode for reading streams of JSON data
	if err := json.NewDecoder(response.Body).Decode(&record); err != nil {
		log.Println(err)
	}

	return record
}
```

## Queue_listener service

Those events will be of no use if they are not consumed. Our service consumes messages from the "events" queue and sends a POST request to the reporting service so the stats can be tracked.

```go
package main

import (
	"log"
	"queue"
)

func main() {

	var nc, err = queue.Connect("server")
	if err != nil {
		log.Fatal(err)
	} else {
		queue.ReceiveMessage(nc, "events")
	}

}

```

# Implementation  

After some quick research I chose GIN as framework to implement the REST endpoints we need to expose for each service.
For the asynchronous communication between *play* and *reporting* we will use a High-Performance server for [NATS ][3], the cloud native messaging system.
I heard about while browsing some PRs from OpenFaas (Function As a service) and decided to give it a go.

Before we start installing our core components, define your **GOPATH** variable to point to the path where your src directory.
The packages will be installed relative to that path, so its libraries can be referenced using import statements.

```bash
export GOPATH=/Users/mfarache/Documents/workspace/go-samples/videoms
```

## Installing Gin

Gin is a HTTP web framework written in Go (Golang) with smashing performance. Visit  [Gin HTTP framework][1] to find more


```bash
CD $GOPATH
go get github.com/gin-gonic/gin
```

## Installing NATS

```bash
CD $GOPATH
#install our NATS server libraries
go get github.com/nats-io/gnatsd
#install our NATS client libraries
go get github.com/nats-io/go-nats
```

## Source code structure and build

The folder structure is as follows

```bash
.
├── cdnurl.go
├── client
│   ├── cdnurlclient.go
│   ├── contentclient.go
│   ├── reportingclient.go
│   └── types
│       ├── contenttype.go
│       ├── playresponsetype.go
│       ├── stattype.go
│       └── urltype.go
├── metadata.go
├── playservice.go
├── queue
│   ├── connect.go
│   ├── publisher.go
│   └── receiver.go
├── queue_listener.go
├── reporting.go
```

As I have not used any Go dependencies system, this script will build everything in one go

```bash
cd $GOPATH/client ; go build ; go install client
cd $GOPATH/client/types ; go build ; go install client/types
cd $GOPATH/queue ; go build ; ; go install queue
cd $GOPATH;
go build metadata.go ;
go build playservice.go ;
go build reporting.go;
go build cdnurl.go ;
go build queue_listener.go
```

## Running all the services

For clarity I opened several iterm windows, and run on each of them one of the commands.
Every microservice uses a default port. It can be override via the --port flag.
Although is a good practice code is still not ready to discover services dynamically so just start them without any parameter.

```bash
#Start our NATS server locally ( By default run on localhost:4222 )
$GOPATH/bin/gnatsd
```

```bash
#Start REPORTING microservice ( will run on localhost:8082 )
$GOPATH/src/reporting
```

```bash
#Start CDNURL microservice ( will run on localhost:8083 )
$GOPATH/src/cdnurl
```

```bash
#Start METADATA microservice ( will run on localhost:8081 )
$GOPATH/src/metadata
```

```bash
#Start PLAY microservice ( will run on localhost:8080 )
$GOPATH/src/playservice
```

```bash
#Start Queue listener microservice ( will connect to NATS server )
$GOPATH/src/queue_listener
```
The following screenshot shows how I lay out my terminal screen where you can see everything in action.

![_config.yml]({{ site.baseurl }}/images/GOMS-TERMINAL.png)

On the left side you can see the output of the 4 services *cdnurl* , *metadata*, *reporting* and *play*.
On the right side on top is our NATS server and in the bottom the *queue_listener*.

So now you can start simulating video viewing actions hitting the play service  and observe how the downstream services are called and stats are generated and consumed via the queu service.

```bash
curl -X POST localhost:8080/play/asset/456/user/8
curl -X POST localhost:8080/play/asset/123/user/1
........
```

# Pending bits

I would had loved to spend time exploring a build system instead of doing go build and go install over and over. You can here [Dependency management with Go][5]

# Summary

The whole code repository is [here][6]

# Useful links

+ [Gin HTTP framework][1]
+ [NATS server. Look for the examples folder][2]
+ [NATS home page][3]
+ [How to write Go code][4]
+ [Dependency management with Go][5]
+ [Github repository for the microservices][6]
+ [Kong API Gateway][7]
+ [Json-to-GO][8]

[1]: https://github.com/gin-gonic/gin
[2]: https://docs.mongodb.com/master/tutorial/enable-authentication/
[3]: https://nats.io/
[4]: https://golang.org/doc/code.html
[5]: https://github.com/golang/dep
[6]: https://mfarache.github.io/mfarache/Building-microservices-with-go/
[7]: https://getkong.org/
[8]: https://mholt.github.io/json-to-go/
