---
layout: post
title: Understanding Kubernetes Pods
tags: [ kubernetes, docker, pods]
---

We will learn what are pods, its lifecycle and several ways of managing them running directly, declaratively using yaml manifests or other alternatives.

# General concepts

A pod is a collection of containers sharing a network and mount namespace that we can deploy in K8.
The containers share namespace, IP and ports. PODs can be reached even across different nodes within the cluster.
Keeping a relation one to one between pod and container is a good idea to reduce coupling.

Pods may contain more than one container, but is recommended to have only one. There are scenarios where we may be interested on having colocated containers running in the same pod, for example a producer, a consumer that share information via same watch folder. We could obviously
deploy two different pods each with its own container... but sometimes life is just easier adding containers together.

The pod itself is not only the container, is also the settings about how it will be run and any additional information regarding volumes storage or networking.

If we deploy more than one container in the same pod, they can communicate using localhost and the port where each of the container is listening.

# Analysis of a pod file

A pod manifest is described in YAML declaratively.
Let´s see with the most simple pod I could imagine. We use this manifest as example in the next section:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox
  namespace: default
spec:
  containers:
  - image: busybox
    command:
      - sleep
      - "3600"
    name: busybox
```

For those familiar with Docker compose, this is the declarative way of saying that we will be using Docker image   busybox and we will execute a command to stay idle for a while.

We said at the beginning that a pod was not only the container, right?
We can define volumes attached to the pod itself. What for? So they can be mounted by the containers.
For each container we can also define ports, mappings, volume mounts & mount paths

The manifest file also support labels attached to metadata of the pod or to the metadata of the container(s) that will run on it.

We are scratching the tip of the iceberg here, so if interested dig deeper in more complex manifest files, although we will see  in the "practical example" section. Something that draw my attention is a setting about the restarting policy that could happen Alway, OnFailure or Never. I do not recall seeing that in Docker Swarm...

# Supervise the pod, yo!

Pods are not meant to live long time nor to be created manually. It is recommended to create the appropriate controller and let it create Pods, rather than directly create Pods yourself.

The real use case scenario is under the supervision of a controller that will schedule the existance of the pod.

The nature of the desired behavior of the pod will help us to determine which kind of controller we need to use. We will see much more in the next blog post which will be focused exclusively in controllers.  A quick summary just to start getting familiar:

+ Do we need a one-off pod that may execute during a finite duration?  Chose Job controller
+ Do we need to keep a long running service?  Chose a Replica or Deployment controller
+ Do we want to target a pod to a specific machine? Chose a Daemon Set.

#  The pod lifecycle

Depending on whether K8 has accepted a request or how the pods terminated (OK vs KO) the pod goes through different status (Pending, Running, Succeeded, Failed and Unknown).

The life of a pod is not intended to last more than the node the pod is attached to. Nodes down in a cluster will evict the pods from the system, or if a scheduler consider that a pod is consuming more resources than expected can be another good reason to see our pod say good-bye and vanish.

In those situation when a pod dies, it dies along the resources attached to it, for example a volume would also be terminated.
The system will assign an UUID that will last till the pod is terminated. In case a node gets down and the pod was under supervision of a replica controller, it will get a new UUID

# Are our pods healthy?

Sometimes we really do not need to check for healthiness. If one of the pods fail by any unexpected reason and depending on the restart policy the main role of the scheduler is restarting the pod again in order to keep our request satisfied.

Imagine our cluster must have 4 pods across the nodes, and one fail. No fear! K8 will detect that situation just because our pod is not attached to any node and according the restarting policy we need to have 4. The self-healing feature does not rely on health check, instead just finds out when the number of pods in our cluster is less than the target specified in the configuration of a controller.

There are other occasion where we want to monitor specific healtchecks. K8 give us the chance to define Pod probes.

What is this? Is just a fancy name to implement health-check. There are different subtypes of handlers via TCP or HTTP that are familiar to anyone who has done monitoring before. I was more interested in the third one (ExecAction) ... Basically command based. Let me elaborate with a simple example:

Let´s imagine our pod has a container that is monitoring a twitter stream of data related to a specific stock value. And let´s also assume that every time it gets a new value, and alert is generated in a folder as a file. Our check could be just based in a linux command executed within the container associated to that pod. The command checks that a new file was added in the last 5 minutes. If the result of the command is not successful something is wrong in our pod..

# Practical example

Ok, let´s work with the previous example

Note: You may need to restart your minikube in local as explained in my previous post [Installing Kubernetes using minikube][5]

```bash
~/Documents/work/k8/pods » kubectl config use-context minikube                              
Switched to context "minikube".

~/Documents/work/k8/pods » minikube start --vm-driver=xhyve                                                                           

Starting local Kubernetes v1.6.4 cluster...
Starting VM...
Moving files into cluster...
Setting up certs...
Starting cluster components...
Connecting to cluster...
Setting up kubeconfig...
Kubectl is now configured to use the cluster.
------------------------------------------------------------
```

At the beginning we described that pods could be just a container or a group of them.
If we just want to create single pod container, is very similar to the docker run where we just specify the image name and mappings for volumes and ports together with environment variables.


```bash
kubectl run example --image=busybox --port=10000 --replicas=5
```
Now we can check they were created succesfully. No mistery here..  5 pods should be created, right?
Well not so fast... as we specified a busybox image, that does not have an entry point, the container automatically exits, crashes and K8 starts again trying to spin up again. See below for different snapshots in time

```bash
NAME                             READY     STATUS             RESTARTS   AGE
example-1830685576-136x9         0/1       CrashLoopBackOff   2          1m
example-1830685576-1jdtk         0/1       Completed          3          1m
example-1830685576-61jdv         0/1       CrashLoopBackOff   2          1m
example-1830685576-fz30h         0/1       Completed          3          1m
example-1830685576-zxr4n         0/1       Completed          3          1m
hello-minikube-938614450-1j5f8   1/1       Running            2          22d
hello-minikube-938614450-bwv2d   1/1       Running            2          22d
hello-minikube-938614450-q5pqm   1/1       Running            2          22d
------------------------------------------------------------
~/Documents/work/k8/pods » kubectl get pods                                                                       
NAME                             READY     STATUS             RESTARTS   AGE
example-1830685576-136x9         0/1       Completed          3          1m
example-1830685576-1jdtk         0/1       CrashLoopBackOff   3          1m
example-1830685576-61jdv         0/1       Completed          3          1m
example-1830685576-fz30h         0/1       Completed          3          1m
example-1830685576-zxr4n         0/1       Completed          3          1m
hello-minikube-938614450-1j5f8   1/1       Running            2          22d
hello-minikube-938614450-bwv2d   1/1       Running            2          22d
hello-minikube-938614450-q5pqm   1/1       Running            2          22d
------------------------------------------------------------
~/Documents/work/k8/pods » kubectl get pods                                                                       
NAME                             READY     STATUS             RESTARTS   AGE
example-1830685576-136x9         0/1       CrashLoopBackOff   4          2m
example-1830685576-1jdtk         0/1       CrashLoopBackOff   4          2m
example-1830685576-61jdv         0/1       CrashLoopBackOff   4          2m
example-1830685576-fz30h         0/1       CrashLoopBackOff   4          2m
example-1830685576-zxr4n         0/1       CrashLoopBackOff   4          2m
```

So lets change the image by nginx and reduce replicas to 3.
Now everything works like a charm, see how the pod status change from creating to Running.

```bash
kubectl run multi-nginx --image=nginx --port=10001 --replicas=3
```

```
~/Documents/work/k8/pods » kubectl get pods                                                                       
NAME                             READY     STATUS              RESTARTS   AGd
multi-nginx-1080823233-17vc3     0/1       ContainerCreating   0          11s
multi-nginx-1080823233-lq59p     0/1       ContainerCreating   0          11s
multi-nginx-1080823233-wv8z5     0/1       ContainerCreating   0          11s
------------------------------------------------------------
~/Documents/work/k8/pods » kubectl get pods                                                                       
NAME                             READY     STATUS              RESTARTS   AGE
multi-nginx-1080823233-17vc3     0/1       ContainerCreating   0          18s
multi-nginx-1080823233-lq59p     0/1       ContainerCreating   0          18s
multi-nginx-1080823233-wv8z5     1/1       Running             0          18s
------------------------------------------------------------
~/Documents/work/k8/pods » kubectl get pods                                  
multi-nginx-1080823233-17vc3     1/1       Running            0          22s
multi-nginx-1080823233-lq59p     1/1       Running            0          22s
multi-nginx-1080823233-wv8z5     1/1       Running            0          22s
```

Now the pods are alive we can use the get command to know more about one of them
Let´s pick the first one:

```bash
~/Documents/work/k8/pods » kubectl get pod multi-nginx-1080823233-17vc3 -o wide          
NAME                           READY     STATUS    RESTARTS   AGE       IP            NODE
multi-nginx-1080823233-17vc3   1/1       Running   0          11m       172.17.0.12   minikube
```
Remember that we are using minikube as one node cluster. If we were using a cluster with multiple nodes K8 would have assigned the pod to any of the nodes available depending on assignment policies or resource quotas.

Before we continue, we will remove all the pods we created using the deployment feature
The outcome of executing run commands with previous examples was the K8 also created a "Deployment" for each of them.
No matter how hard we try to remove the pods individually, the replica controller associated will try to create them again.
So the only solution is to wipe out the deployment itself and then the pods will die gracefully

```bash
kubectl delete deployment example
kubectl delete deployment multi-nginx   
```

We can also create pods using manifest fields describing the container(s) that make the pod.

we will grab the example at the begininng and store in our filesystem with busybox.yml.

```bash
~/Documents/work/k8/pods » kubectl create -f busybox.yml                                                                  
pod "busybox" created
------------------------------------------------------------
~/Documents/work/k8/pods » kubectl get pods                         
NAME      READY     STATUS    RESTARTS   AGE
busybox   1/1       Running   0          4s
------------------------------------------------------------
```
It has created our "sleeping" container during one hour. Then it will die.
Why? Remember in our manifest the command entry was just a simple "sleep 3600"

For illustrative purposes we can see other example, slightly more complex.
It will run cadvisor (one of the tools I use in case I need to troubleshoot performance issues).
Those familiar with docker compose should find it very similar... A few volume and volume mounts, one on /var/lib/docker so it can monitor the events that occurr in docker itself. You also can see that there is a port mapping from 8080 to hostPort that would our access entry point.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cadvisor-agent
spec:
  containers:
    - name: cadvisor-agent
      image: google/cadvisor:0.19.3
      ports:
        - name: http
          containerPort: 8080
          hostPort: 4194
      volumeMounts:
        - name: varrun
          mountPath: /var/run
          readOnly: false
        - name: varlibdocker
          mountPath: /var/lib/docker
          readOnly: true
        - name: sysfs
          mountPath: /sys
          readOnly: true
  volumes:
    - name: varrun
      source:
        hostDir:
          path: /var/run
    - name: varlibdocker
      source:
        hostDir:
          path: /var/lib/docker
    - name: sysfs
      source:
        hostDir:
          path: /sys
```

To complete our deep dive into pods , just mentioning that there are more ways to create pods:

+ You could post the manifest via API to achieve the same result. The only gotcha is that the API server only accepts manifest in JSON format, so you would need to convert from YAML into JSON.
+ You could drop the manifest file in a watch folder monitored by K8. The pod will be scheduled creation without human intervention.

# Useful links

+ [Static pods][1]
+ [Multicontainer pods][2]
+ [Kubernetes from the group up to the API server][3]

[1]: https://kubernetes.io/docs/tasks/administer-cluster/static-pod/
[2]: https://lukemarsden.github.io/docs/user-guide/pods/multi-container/
[3]: http://kamalmarhubi.com/blog/2015/09/06/kubernetes-from-the-ground-up-the-api-server/

### The whole blog series about Kubernetes

+ [1. Introduction Kubernetes vs Swarm.][4]
+ [2. Installing Kubernetes using minikube][5]
+ [3. Understanding Kubernetes pods.][6]
+ [4. Understanding Kubernetes controllers.][7]
+ K8 services to access pods

[4]: https://mfarache.github.io/mfarache/Introduction-Kubernetes-compared-Swarm/
[5]:https://mfarache.github.io/mfarache/Installing-Kubernetes-using-Minikube/
[6]:https://mfarache.github.io/mfarache/Understanding-Kubernetes-Pods/
[7]:https://mfarache.github.io/mfarache/Understanding-Kubernetes-Controllers/
