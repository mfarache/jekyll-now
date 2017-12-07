---
layout: post
title: Functions as a service with OpenFaas
tags: [ OpenFaaS, Serverless, Kubernetes, Functions As A Service ]
---

OpenFaaS is a framework for packaging code, binaries or containers as Serverless functions on any platform.
![_config.yml]({{ site.baseurl }}/images/OPENFAAS_LOGO.png)

# Motivation

After getting back from Amazon Reinvent 2017 I wrote a blog entry about AWS Lambdas.

For at least 5 months I have followed  progress in twitter about @alexellisuk (Alex Ellis) who is a Docker Captain.
He has done pretty cool things around containers and crazy Raspberry docker clusters in the past.

He started OpenFaas around one year ago when he wanted to understand if he could run Lambda functions on Docker Swarm.

I had visited his Github repo a few times but never had the chance to play with OpenFaas.

Openfaas provides support for Docker Swarm and Kubernetes (recently added faas-netes).
As lately I have been playing with K8 and is fresh in my mind I thought there were no excuses to have a bit of fun.

# Openfaas features

Functions as a service is one of the trending topics together with "serverless" architectures.
Evolution from monolith to microservice was the first step of a drastic change.
Now we need to start speaking about Functions as minimum deployable units that perform a single action.

Openfaas provides :

+ Easy install (1minute!)
+ Multiple language support
+ Transparent autoscaling
+ No infrastructure headaches
+ Support for Docker Swarm and Kubernetes
+ Console UI to enable deployment, invocation of functions
+ Integrated with Prometheus Alerts
+ Async support
+ Market Serverless function repository
+ RESTful API
+ CLI tools to build & deploy functions to the cluster.

# Installation of OpenFaas

The following steps require MAC, K8, local minikube

In case of doubts about minikube review this entry [Installing Kubernetes using minikube][5]

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

Done!


Before we move to the next step, we need to get the IP address of our Kubernete node so we can  access to its API gateway and its UI.

```bash
» minikube ip                                                                                                       There is a newer version of minikube available (v0.24.1).  Download it here:
https://github.com/kubernetes/minikube/releases/tag/v0.24.1

To disable this notification, run the following:
minikube config set WantUpdateNotification false
192.168.64.2
```
We can check that UI works correctly hitting http://192.168.64.2:31112/ui/  (replace with your own K8 IP address)

![_config.yml]({{ site.baseurl }}/images/OPENFAAS_UI.png)

# Installation of OpenFaas CLI

The client allows building and deploying functions as a service based on a yaml descriptor
Let´s deploy the samples provided

```bash
» brew install faas-cli
» git clone https://github.com/alexellis/faas-cli
```
# Deployment of functions

There are some samples on the repositoty that gives us a chance to play with deployment of functions.

```bash
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
» faas-cli list -f samples-node-minikube.yml                                                                       

Function                      	Invocations    	Replicas
nodejs-echo                   	0              	1
ruby-echo                     	0              	1
shrink-image                  	0              	1
stronghash                    	0              	1
url-ping                      	0              	1
```
# Understanding definition of functions

Our functions only need to deal with STDIN and STDOUT.
We can interact with our functions via an API Gateway which maps incoming requests to the input processed by our function. The output of our process is marshalled back as response to our API gateway call.

OpenFaas supports ANY function as far as a Docker image is provided.

Adding new function is pretty simple, via YAML file.

## 1. Functions defined as language handlers

Our function can be an entry point defined by a language handler (Python, Ruby, Nodejs)

To get a better understanting we can analyze the contents of the functions section within our samples file.

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

## 2. Functions defined as processes

Our Dockerfile should read a fprocess enviroment variable and define an entrypoint with this command.
The following example shows how the variable is defined as a imagemagick command.

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
# OpenFaas REST API

The CLI is not the only way to interact with OpenFaaS.

For the curl fan-addicted a restful API is available exposing CRUD operations on functions, deployments and status check

![_config.yml]({{ site.baseurl }}/images/OPENFAAS_SWAGGER.png)

We can invoke our functions using http://<K8IPADDRESS>:<K8PORT>/function/<FUNCTIONNAME>

If we wanted to call to the NodeJS echo function that we saw in the previous section
```bash
» curl http://192.168.64.2:31112/function/nodejs-echo -d "Test"
```

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

I opted to run Graphana, create a DataSource pointing to Prometheus

```bash
docker run -d --name=grafana -p 3000:3000 grafana/grafana
```
Once is running we just need to create a datasource and create a panel where we can see the metrics.

In Grahana console:
 + DataSources
 + Add Datasource
 + Select Prometheus
 + Type the  URL: http://192.168.64.2:31119/
 + Select direct access

We will import the following Graphana panel (https://grafana.com/dashboards/3434) aimed to monitor openFaas

In Grahana console:
  + Select Dashboard
  + Select Import panel
  + Enter the ID3434
  + Aassociate to the DataSource we created in the previous steps (openfaas-prometheus-ds)

After all this I cannot see any metric :(

At the moment of writing this post, it seems a migration to Prometheus 2.0 could be the culprit. This PR https://github.com/openfaas/faas/pull/412 blocked currently, states that on Kubernetes metrics does not work, so really looking forward to see this issue fixed.

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
As OpenFaas has bee designed with extensibility in mind, other implementation will come in the future. For starters browsing github I found there is change request to support AWS SNS topics.

# Other alternatives

It's worthy mentioning that there are other frameworks out there like openwhisk, funker or iron functions but none of them run on K8.
If we want to compare with frameworks that support K8 would be looking at

+ Kubeless (really just a POC)
+ Funktion (too tight coupling with Fabric, only one language python, reuse of camel connectors)
+ Fision   (Written in Go, only supported languages are Node & Python, based on http triggers)

I really like the extensible architecture and design principles of Openfaas. Also the project has rocketed in the last months wiht forks, stars (on the more trending github Go project) and user contributors so I think OpenFaas is here to stay.

Keep up the good work, Alex!

# Useful links

+ [OpenFaaS][1]

[1]: https://www.openfaas.com/
[4]: https://mfarache.github.io/mfarache/Introduction-Kubernetes-compared-Swarm/
[5]:https://mfarache.github.io/mfarache/Installing-Kubernetes-using-Minikube/
[6]:https://mfarache.github.io/mfarache/Understanding-Kubernetes-Pods/
[7]:https://mfarache.github.io/mfarache/Understanding-Kubernetes-Controllers/
