---
layout: post
title: Serverless function backed by database using OpenFaaS
tags: [ Serverless, OpenFaaS ]
---

Using OpenFaas deployed into Docker Swarm to deploy a NodeJS serverless function backed by a MongoDB database

# The idea

After getting started with OpenFaas and Kubernetes in my previous post, now it was time to test it with Docker Swarm.
In order to spice up things a little , I decided that the serverless function could have a external dependency like a database

Url shortening is a perfect candidate to be implemented as a serverless function. I will focus just in two main uses cases

As a user
I want to shorten a URL
So I can share the shortened URL via twitter

As a user
I want to resolve a shortened URL
So I can visit the URL associated to the shortened one

If we want to implement both scenarios we need to persist both the URL and the shortened URL so it can be resolved later.
We will use a No-SQL document database like MongoDb to do the job.

Each of the use cases will be implemented as a NodeJS function deployed into OpenFaas.

The following diagram shows what we will build.

![_config.yml]({{ site.baseurl }}/images/OPENFAAS-MONGO.png)


# OpenFaas and Docker Swarm

In previous post I explained how to deploy using Kubernetes, however in this post I will deploy OpenFaas using Docker Swarm.
I will assume you have latest Docker version and Docker swarm enabled

```
docker swarm init
```

We will use a YAML file to deploy the whole stack.

```yaml
version: "3.2"
services:
    gateway:
        volumes:
            - "/var/run/docker.sock:/var/run/docker.sock"
        ports:
            - 8080:8080
        image: functions/gateway:0.6.13
        networks:
            - functions
        environment:
            dnsrr: "true"  # Temporarily use dnsrr in place of VIP while issue persists on PWD
        deploy:
            placement:
                constraints:
                    - 'node.role == manager'
                    - 'node.platform.os == linux'
    prometheus:
        image: functions/prometheus:latest  # autobuild from Dockerfile in repo.
        command: "-config.file=/etc/prometheus/prometheus.yml -storage.local.path=/prometheus -storage.local.memory-chunks=10000 --alertmanager.url=http://alertmanager:9093"
        ports:
            - 9090:9090
        depends_on:
            - gateway
            - alertmanager
        environment:
            no_proxy: "gateway"
        networks:
            - functions
        deploy:
            placement:
                constraints:
                    - 'node.role == manager'
                    - 'node.platform.os == linux'

    alertmanager:
        image: functions/alertmanager:latest    # autobuild from Dockerfile in repo.
        environment:
            no_proxy: "gateway"
#        volumes:
#            - ./prometheus/alertmanager.yml:/alertmanager.yml
        command:
            - '-config.file=/alertmanager.yml'
        networks:
            - functions
        ports:
            - 9093:9093
        deploy:
            placement:
                constraints:
                    - 'node.role == manager'
                    - 'node.platform.os == linux'

    mongodb:
        image: mongo:3.4.10
        networks:
            - functions
        ports:
            - 27017:27017
        volumes:
            - '/datastore/mongodb:/data/db'
        environment:
            no_proxy: "gateway"
            https_proxy: $https_proxy
            # provide your credentials here
            MONGO_INITDB_ROOT_USERNAME: "urlshortener"
            MONGO_INITDB_ROOT_PASSWORD: "urlsh0rt3n3r"
        deploy:
            placement:
                constraints:
                    - 'node.platform.os == linux'

    urlshortener:
        image: dockermau/openfaas-mongodb-urlshortener:1.0
        labels:
            function: "true"
        depends_on:
            - gateway
            - mongodb
        networks:
            - functions
        environment:
            no_proxy: "gateway"
            https_proxy: $https_proxy
        deploy:
            placement:
                constraints:
                    - 'node.platform.os == linux'

networks:
    functions:
        driver: overlay
        # Docker does not support this option yet - maybe create outside of the stack and reference as "external"?
        #attachable: true
```
We define in the stack services

+ OpenFaas main component: the gateway
+ OpenFaas functions: alertmanager and prometheus.
+ Our mongodb image (See note below about how to enable auth)
+ Our serverless function, which refers to an image in my personal Docker registry

# MongoDB setup

Before we go ahead with the definition, a note about authentication with MongoDB
By default authentication is not enabled so there are some required steps we need to follow.
If we want to use the mongoDB image with authentication enabled the steps to follow are:

```
docker run  --name mongodb -v /datastore/mongodb:/data/db -e MONGO_INITDB_ROOT_USERNAME="urlshortener" -e MONGO_INITDB_ROOT_PASSWORD="urlsh0rt3n3r" mongo:3.4.10
```

As we passed the parameters , the user now it has been created.
Now we can safely stop the container
```
docker stop mongodb
```
You can see that  I run the container with a mapping volume. This way the changes done to the database are kept in the mapped directory of the host. And you can also realize that the mapped volume matches with the one defined in our docker-compose file.
So once the stack is launched, MongoDB would allow authentication and our function will be able to connect to the mongo instance.

# Our OpenFaaS function

Reviewing samples from OpenFaaS github repositories, it seems that a  good practice is breaking down your code in two parts

index.js: Which mainly read from STD input and delegates processing to a handler function which does the job.

```js
"use strict"

let getStdin = require('get-stdin');
let handler = require('./function/handler');

getStdin().then(val => {
    handler(val, (err, res) => {
        if (err == null) {
          if(isArray(res) || isObject(res)) {
              console.log(JSON.stringify(res));
          } else {
              console.log(res);
              process.exit(0);
          }
        } else {
          if (err) {
              return console.error(err);
          }
        }
    });
}).catch(e => {
    console.error(e.stack);
});

let isArray = (a) => {
    return (!!a) && (a.constructor === Array);
};

let isObject = (a) => {
    return (!!a) && (a.constructor === Object);
};
```

handler.js

I used short library that does the job of shortening our url and storing them against a mongodb schema via promises.
It´s very important not to use conole.log after the "shortURLPromise.then" statement.
Why? If we do so, the caller of our callback context (index.js) would think is the response.

```js
"use strict"

// Require Logic
var short = require("short")
var log4js = require('log4js');
log4js.configure({
  appenders: { 'file': { type: 'file', filename: 'urlshortener.log' } },
  categories: { default: { appenders: ['file'], level: 'debug' } }
});

var logger = log4js.getLogger('urlshortener');

//local testing
//var mongoDbUrl =`mongodb://urlshortener:urlsh0rt3n3r@localhost:27017/admin`;

var mongoDbUrl =`mongodb://urlshortener:urlsh0rt3n3r@mongodb:27017/admin`;

module.exports = (context, callback) => {
    const url = JSON.parse(context);
    // connect to mongodb
    logger.debug ( "url:" + url);
    logger.debug("mongo:" + mongoDbUrl);
    short.connect(mongoDbUrl);

    logger.debug("context: " + context);

    short.connection.on('error', function(error) {
        callback(error, null);
    });
    // promise to generate a shortened URL.
    //context should be a JSON doc like { URL: "http://wwww.someurl.com"}
    var shortURLPromise = short.generate({ URL: url.URL});

    // gets back the short url document, and then retrieves it
shortURLPromise.then(function(mongodbDoc) {
  logger.debug("shorten:" + mongodbDoc.hash);
  callback(null, mongodbDoc.hash);
}, function(error) {
  if (error) {
    logger.debug(error);
    callback(error, null);
  }
});
}
```

Before we build our function we can test locally, restarting the container we stopped in previous steps, and be sure that in handler.js the connection string url refers to localhost instead.

Note:  MongoDB URL is hardcoded, however the code should be modified to retrieve the host, port, username and password from the environment variables which are poopulated from the docker-compose.yaml file

```bash
node index.js < in.text
```

where in.txt looks like

```JSON
{
  "URL" : "http://www.google.com"
}
```


Once we know our function code works locally, we can revert the change to use the proper URL.

The following steps shows how I built my code function and pushed to my own Docker registry.
If you intend to do any changes you should consider using your own registry and modify the docker-compose file so the function refers to your image.

If we want to see everything in action

```bash
git clone
cd openfaas-mongodb-urlshortener
#See the section about enable authorization and to add the user to the database
./deploy_stack.sh
```

After deploying our stack we can see our two functions running as docker containers

```bash
» docker ps     
CONTAINER ID        IMAGE                                         COMMAND                  CREATED             STATUS                       PORTS                    NAMES
ca627f311b37        dockermau/openfaas-mongodb-urlresolver:1.0    "fwatchdog"              28 minutes ago      Up 28 minutes (healthy)                               func_urlresolver.1.jrmkvuolxolw2innzpcpz7xod
1b9979234432        dockermau/openfaas-mongodb-urlshortener:1.0   "fwatchdog"              About an hour ago   Up About an hour (healthy)                            func_urlshortener.1.k559f18qza0sta6tihy7hb6ny
7955adb623a9        functions/gateway:0.6.13                      "./gateway"              6 hours ago         Up 6 hours                   8080/tcp                 func_gateway.1.qkid168rxgjpmaw4ziqodpl21
fcf142a8bdda        functions/alertmanager:latest                 "/bin/alertmanager..."   19 hours ago        Up 19 hours                  9093/tcp                 func_alertmanager.1.xp103sq9w5pvq47q82rulri5q
5f34f8c6df61        functions/prometheus:latest                   "/bin/prometheus -..."   19 hours ago        Up 19 hours                  9090/tcp                 func_prometheus.1.cqhwocnf02xgj36qttln9z44o
197b4565811b        mongo:3.4.10                                  "docker-entrypoint..."   19 hours ago        Up 19 hours                  27017/tcp                func_mongodb.1.kihxayta8j5eno1l6fm875ucn
```

We can see our functions in action hitting the OpenFaaS UI ( http://localhost:8080/UI) for those working locally.

First let's shorten a URL

![_config.yml]({{ site.baseurl }}/images/OPENFAAS-URLSHORTENER.png)

Then let's see if is able to resolve the previous shortened URL

![_config.yml]({{ site.baseurl }}/images/OPENFAAS-URLRESOLVER.png)

#Summary

The more I play with OpenFaas the more I like it.  
It took me a few hours to setup everything and running.
The bits I really struggled with where related with MongoDB authentication.
It also was not easy to debug what was going on within the NodeJS functions running on top of OpenFaaS, but luckily there are some awesome tips in the community issues that helped me to dig out the bottom of the issues. Some tips for newbies like me

+ Add log4js to see what is going within your NodeJS function.
+ Our index.js needs to indicate process.exit(0) so the gateway considers our request to be completed. Otherwise the action will be performed but you will never get back the response.

The whole code repository is [here][3]

# Useful links

+ [short nodejs package][1]
+ [enable authentication on MongoDB][2]
+ [URL shortener openfaas github repository][3]
+ [Remote Debugging openFaas apps][4]
+ [Improve logging for functions during erorr][5]
[1]: https://www.npmjs.com/package/short
[2]: https://docs.mongodb.com/master/tutorial/enable-authentication/
[3]: https://docs.mongodb.com/master/tutorial/enable-authentication/
[4]: https://github.com/openfaas/faas/issues/223
[5]: https://github.com/openfaas/faas/issues/11
