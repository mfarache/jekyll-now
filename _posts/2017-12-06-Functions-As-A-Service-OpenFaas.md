---
layout: post
title: Functions as a service with OpenFaas
tags: [ OpenFaaS, Serverless, Functions As A Service ]
---

OpenFaaS is a framework for packaging code, binaries or containers as Serverless functions on any platform.
![_config.yml]({{ site.baseurl }}/images/OPENFAAS_UI_DEPLOY.png)

# Motivation

After getting back from Amazon Reinvent 2017 I wrote a blog entry about AWS Lambdas.

For at least 5 months I have followed  progress in twitter about @alexellisuk (Alex Ellis) who is a Docker Captain.
He has done pretty cool things around containers and crazy Raspberry docker clusters in the past.

He started OpenFaas as a proof of concept in 2016 October, when he wanted to understand if he could run Lambda functions on Docker Swarm.

I had visited his github repo a few times but never had the chance to play with OpenFaas.

Recently I heard that he added support for K8 via a new project, so now that K8 is fresh in my mind I thought there were no excuses to have a bit of fun.

# Openfaas features

Functions as a service is one of the trending topics together with "serverless" architectures.
Evolution from monolith to microservice was the first step of a drastic change.
Now we need to start speaking about Functions as minimum deployable units that perform a single action.

Openfaas provides :

+ Easy install (1minute!)
+ Multiple language support
+ Transparent autoscaling
+ No infrastructe headaches
+ Support for Docker Swarm and Kubernetes
+ Console UI to enable deployment, invocation of functions
+ Integrated with Prometheus Alerts
+ Asyn support
+ Market-like serverless function repository
+ RESTful API
+ CLI tools to build & deploy functions to the cluster.

# Installation of OpenFaas

We will use Kubernetes as deployment method for OpenFaas.
We will use minikube running on a Mac from now onwards.

In case of doubts about minikube or K8 I have a bunch of posts that you may find interesting
+ [1. Introduction Kubernetes vs Swarm.][4]
+ [2. Installing Kubernetes using minikube][5]
+ [3. Understanding Kubernetes pods.][6]
+ [4. Understanding Kubernetes controllers.][7]

Deployment of OpenFaas is as easy as cloning a git repo and running a single command.

```bash
»git clone https://github.com/openfaas/faas-netes
»cd faas-netes
»kubectl apply -f ./faas.yml,monitoring.yml,rbac.yml

service "faas-netesd" created
serviceaccount "faas-controller" created
deployment "faas-netesd" created
service "gateway" created
deployment "gateway" created
service "prometheus" created
deployment "prometheus" created
service "alertmanager" created
deployment "alertmanager" created
clusterrole "faas-controller" created
clusterrolebinding "faas-controller" created
```

Before we move to the next step, we need to get the IP address of our Kubernete node so we can  access to its API gateway and its UI.

```bash
» minikube ip                                                                                                       mfarache@OEL0043
There is a newer version of minikube available (v0.24.1).  Download it here:
https://github.com/kubernetes/minikube/releases/tag/v0.24.1

To disable this notification, run the following:
minikube config set WantUpdateNotification false
192.168.64.2
```
We can check that UI works correctly hitting http://192.168.64.2:31112/ui/  (replace with your own K8 IP address)

![_config.yml]({{ site.baseurl }}/images/OPENFAAS_UI.png)

#Installation of OpenFaas CLI

The client allows building and deploying functions as a service based on a yaml descriptor
Let´s deploy the samples provided

```bash
#On a MAC
» brew install faas-cli
» git clone https://github.com/alexellis/faas-cli
» cd faas-cli
» sed "s/localhost:8080/192.168.64.2:31112/g" samples.yml > samples-node-minikube.yml
» faas-cli deploy -f samples-node-minikube.yml

Deploying: shrink-image.
No existing function to remove
Deployed.
URL: http://192.168.64.2:31112/function/shrink-image

202 Accepted
Deploying: ruby-echo.
No existing function to remove
Deployed.
URL: http://192.168.64.2:31112/function/ruby-echo

202 Accepted
Deploying: url-ping.
No existing function to remove
Deployed.
URL: http://192.168.64.2:31112/function/url-ping

202 Accepted
Deploying: stronghash.
No existing function to remove
Deployed.
URL: http://192.168.64.2:31112/function/stronghash

202 Accepted
Deploying: nodejs-echo.
No existing function to remove
Deployed.
URL: http://192.168.64.2:31112/function/nodejs-echo

202 Accepted
```
Once the functions have been deployed we can see their status. None of the functions has been ever invoked.

```bash
» faas-cli list -f samples-node-minikube.yml                                                                       mfarache@OEL0043
Function                      	Invocations    	Replicas
nodejs-echo                   	0              	1
ruby-echo                     	0              	1
shrink-image                  	0              	1
stronghash                    	0              	1
url-ping                      	0              	1
```

OpenFaas uses STDIN to send metadata parameters to our function and STDOUT as mechanism  to deliver its output.
That is the perfect combination to use linux pipeline commands.

The framework supports ANY function as far as a Docker image is provided.
When defining a function can be either entry point defined by a language handler or entrypoint defined by process launched on docker.
To get a better understanting we can analyze the contents of the functions section within samples-node-minikube.yml

In this snippet we see that the function is based in an image which will invoke the python handler located in the relative directtory ./sample/url-ping
```yaml
url-ping:
    lang: python
    handler: ./sample/url-ping
    image: alexellis/faas-url-ping
    limits:
      memory: 10m
    requests:
      memory: 10m
```
The convention is handler.py for python , handler.rb for Ruby and handler.js for NodeJS

However in the next step  we see that the function is based in an image which will execute a specific command line. This specific image brings the imagemagick tools in order to provide image resizing functions.

How is this implemented? fprocess is a environment variable that is used by the image built by the Dockerfile and is the entry execution point.

```yaml
shrink-image:
   lang: Dockerfile
   handler: ./sample/imagemagick
   image: functions/resizer
   fprocess: "convert - -resize 50% fd:1"
   limits:
     memory: 20m
   requests:
     memory: 20m
```

We can try few examples in Ruby, Python and Node.
Be sure to append the --gateway flag to override the default settings.

```bash
-----------------------------------------------------------
» echo -n Test | faas-cli invoke stronghash --gateway http://192.168.64.2:31112
c6ee9e33cf5c6715a1d148fd73f7318884b41adcb916021e2bc0e800a5c5dd97f5142178f6ae88c8fdd98e1afb0ce4c8d2c54b5f37b30b7da1997bb33b0b8a31  -
------------------------------------------------------------
» echo -n Test | faas-cli invoke nodejs-echo --gateway http://192.168.64.2:31112
{"nodeVersion":"v6.11.2","input":"Test"}
------------------------------------------------------------
» echo -n Test | faas-cli invoke ruby-echo --gateway http://192.168.64.2:31112
Hello from your Ruby function. Input: Test
------------------------------------------------------------
» echo -n "http://www.google.com" | faas-cli invoke url-ping --gateway http://192.168.64.2:31112
Handle this -> http://www.google.com
http://www.google.com => 200
```

The CLI is not the only way to interact with OpenFaaS.
For the curl fan-addicted a restful API is available exposing CRUD operations on functions, deployments and status check

![_config.yml]({{ site.baseurl }}/images/OPENFAAS_SWAGGER.png)

# OpenFaas Market

Using the UI we can also deploy functions from a online market. I will deploy the Text-To-Speech generator

![_config.yml]({{ site.baseurl }}/images/OPENFAAS_DEPLOY.png)

Now we can invoke it using the CLI invoke command

```bash
» echo -n "OpenFaas is an awesome framework" | faas-cli invoke text-to-speech  --gateway http://192.168.64.2:31112 > ~/Documents/workspace/faas-output.mp3
```

You can also use the API to achieve the same results

```bash
» curl http://192.168.64.2:31112/function/text-to-speech -d "OpenFaas is an awesome framework" > ~/Documents/workspace/faas-output.mp3
```
Works like a charm and the file reproduces the text I expected to hear.

# Prometheus

OpenFaas uses an API gateway that collects user metrics available in Prometheus.
Prometheus  :http://192.168.64.2:31119/

# Autoscaling

The gateway also scale the functions based on demand of your containers.

We can test autoscaling just invoking the function multiple times in a infinite loop.
In order not to starve our resource we will sleep 0.2 secs before we launch a new call.
Note that the body of the while loop is run in background. In these conditions I would expect that the number of replicas of the function would start growing after some warm-up time... however it did not :( and not a clue why not.

Reducing the time to 0.1sec did not help. In less than 1 minute I had launched 300 requests and openFaas become irresponsive.
```bash
while [ true ] ; do sleep 0.1 && echo -n "OpenFaas is an awesome framework" | faas-cli invoke text-to-speech  --gateway http://192.168.64.2:31112 > ~/Documents/workspace/faas-output.mp3 & ; done
```

As the test implied writing into a MP3 file I thought the issue could be concurrent lock on the file itself.
As the purpose of the test is just see how replica works I changed the approach to use one of the echo services

```bash
while [ true ] ; do sleep 0.3 && echo -n Test | faas-cli invoke nodejs-echo --gateway http://192.168.64.2:31112 & ; done
```
Same result.

I found this bug that could explain it: https://github.com/openfaas/faas-netes/issues/36
I would love to see the auto-scaling in action.

# Async process

Currently there is a NATS implementation but there is a PR that aims to deliver Kafka as a mechanism for the streaming of events into OpenFaas.
Any function as a service

# Other alternatives

It's worthy mentioning that there are other frameworks out there like openwhisk, funker or iron functions but none of them run on K8.
If we want to compare with frameworks that support K8 would be looking at

+ Kubeless (really just a POC)
+ Funktion (too tight coupling with Fabric, only one language python, reuse of camel connectors)
+ Fision   (Written in Go, only supported languages are Node & Python, based on http triggers)

I really like the architecure design principles of Openfaas. Also the project has rocketed in the last months wiht forks, stars (on the more trending github Go project) and user contributors so I think OpenFaas is here to stay. Keep up the good work, Alex!

# Useful links

+ [OpenFaaS][1]

[1]: https://www.openfaas.com/
[4]: https://mfarache.github.io/mfarache/Introduction-Kubernetes-compared-Swarm/
[5]:https://mfarache.github.io/mfarache/Installing-Kubernetes-using-Minikube/
[6]:https://mfarache.github.io/mfarache/Understanding-Kubernetes-Pods/
[7]:https://mfarache.github.io/mfarache/Understanding-Kubernetes-Controllers/
