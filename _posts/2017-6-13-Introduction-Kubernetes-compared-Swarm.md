---
layout: post
title: Kubernetes vs Swarm
tags: [ kubernetes, docker]
---

This post will go through a brief introduction to K8 and see how it stands against Docker Swarm. 

#  Why?
I used Docker Swarm just for play in my laptop, sping up a few nodes in the cluster and everything works like a charm.
After reviewing some feature comparison with K8 all of them  indicate very similar or not benefit.
 Speaking with some colleagues they told me it would be crazy to run Docker swarm in production and I wanted to know why.
I´m sure I was missing something, so I wanted to know the "goodies" of K8. So as introduction step, the best thing is understanding the main concepts, how it works and try to get my hands dirty. I will start a series on Kubernetes and this will be the introduction to the main concepts as I learn them, so be benevolent if I say something stupid.

# .. so what is Kubernetes?

Kubernetes , AKA K8 , allow us to deploy and manage containers at will across several hosts within a cluster. Still do not see a difference with Swarm.. let´s find out all the hype about it. We also can schedule and run application containers on clusters of physical or virtual machines.

Apparently there is no restriction on the type of aplication (stateless, stateful, batch  data-processing). If an application can run in a container, it should run great on Kubernetes.

The list of features are:

+ Mounting storage systems
+ Distributing secrets
+ Checking application health
+ Replicating application instances
+ Using Horizontal Pod Autoscaling
+ Naming and discovering
+ Balancing loads
+ Rolling updates
+ Monitoring resources
+ Accessing and ingesting logs
+ Debugging applications
+ Providing authentication and authorization

So far so good, basic concepts that are also included in Docker Swarm. :(
The search for the shining features goes on..

# Kubernet architecture

There are many articles with architecture description. I found this diagram summarizes everything I need to know now.

![_config.yml]({{ site.baseurl }}/images/K8_ARCH.png)

# Kubernet concepts

I tend to be impatient when I want to learn something new. Generally I just find for the getting started guide, the less invasive for my lappy and start trying. However ..as the whole thing is pretty new, I thought this time a first walk through across concepts and some diagram  would help me understand the terminology of K8 main components.

![_config.yml]({{ site.baseurl }}/images/K8_OVERVIEW.png)

## PODS
A pod is a collection of containers sharing a network and mount namespace that we can deploy in K8.
The containers share namespace, IP and ports. PODs can be reached even across different nodes within the cluster.
Keeping a relation one to one between pod and container is a good idea to reduce coupling.

## Labels
A semantic tagging that applies to PODs. Useful when scheduling pods across nodes via selectors, for rolling updates, specific targeting,etc.

## Service
It provides a virtual IP adress and a DNS for a set of pods.
You can see it as a traffic forwarder.  Since Pods can be created and destroyed dinamically, the service gives an abstraction to access using the VIP (regardless the node).
It uses IPTables  to achieve that. They keep track of the pods, and forwards traffic using the label
You can compare with a addressable load balancer with random pod assignment strategy

## Deployment
Supervises pods and replica sets. We can define parameters like how many replicas and which pods are involved in the container. To do that we define the container(s) very similar way as we would do in docker compose. Deployments are specified via YAML files.
The following is a deployment example

```yaml

apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: sise-deploy
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: sise
    spec:
      containers:
      - name: sise
        image: mhausenblas/simpleservice:0.5.0
        ports:
        - containerPort: 9876
        env:
        - name: SIMPLE_SERVICE_VERSION
          value: "0.9"
```

## Replication Controllers
Is like the police vigilant for long-running pods. Any pod replica must stay alive running.
Each RC use labels to track the pods he is responsible for.

## Nodes
It´s where stuff happens. All of them belong to a K8 cluster. It´s Just the machine where your pod is deployed.
We can tag nodes with labels. And we can instruct K8 to create a pod to be scheduled on a node with a specific label.

## Jobs
Supervise pods that have a finite lifetime.

## Replica Sets
Responsable of keeping a number of pods running ( spinning up & down pods to reach the target).
Replica sets allow defining scaling strategies grouping by label selectors, canary deployment or autoscaling based on cpu usage.

So what really seems interesting is the feature one replica sets. See some examples which are self-explanatory.

![_config.yml]({{ site.baseurl }}/images/K8_AUTOSCALE.png)

![_config.yml]({{ site.baseurl }}/images/K8_CANARY.png)

At least now this is the first thing which clearly sees a benefit when compared with Docker Swarm, and really a differentiator!

Docker Swarm has other disadvantages vs K8 like
+ is not possible to reschedule nodes
+ horizontal autoscaling setup seems tricky,
+ health checks (wip)
+ not as mature as K8, every release has some issues

Obviusly there are more advanced concepts such as volumes, namespaces, secret management, healthchecks, etc... but I think that so far we enough new concepts to start with, so we will get our hands dirty in the next post. We will use the recommended K8 minikube for shake of simplicity using a one node cluster.
