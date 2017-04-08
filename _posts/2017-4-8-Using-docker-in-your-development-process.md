---
layout: post
title: Build your JAVA code with maven within a Docker container
tags:   [ java, maven, docker ]
---
The following post will show how developers could benefit of using Docker on the development cycle.
Docker can be used not only to improve deployment processes or to be part of

# Local builds
Development teams are made of different mindsets and preference so is natural that everyone run his favourite set of tools on top of their operative system using Windows
Mac Os, or any of the linux distro variants. Sometimes we face the issue that a build works in a laptop, but not in other or even worse ... how many times have we heard
My local build works fine! but in our CI environment using Jenkins fails..

To avoid such scenario we have introduced recently a "dockerized" build approach. The benefits are obvious
+ Everyone has exactly the same build environment, i.e JDK version, MVN version.
+ The developer´s build environment is exactly the same as the one used by our CI server
+ A new starter can join to the team and start building straight away

The first approach was quite naive
```
docker run -it --rm -w /opt/maven -v $PWD:/opt/maven  maven:3.3-jdk-8 mvn clean install
```
As the image is pretty new it would download the whole internet due to maven dependencies. If we rebuild or we have to remove the container and start it again it would be a painful waste of time.

The next iteration I used a volume that I can easily attach to the container, so event stoping/restarting it would keep the Maven repository available

```
docker run -it --rm -w /opt/maven -v $PWD:/opt/maven -v $HOME/.m2:/root/.m2 maven:3.3-jdk-8 mvn clean install
```

Event that way on my Mac OsX  realized that building this way is ridicolously slow. The reason is not that the container is downloading the whole internet as result of the maven dependencies, because we start our container with a volume mapped to the developer host. If the developer had built the project unless once, the container would have exactly the same $MAVEN_HOME repository fully available.

Let´s see what is going on our image. We will use Google´s cadvisor image that will give us some insights stats.

```
sudo docker run \
  --volume=/:/rootfs:ro \
  --volume=/var/run:/var/run:rw \
  --volume=/sys:/sys:ro \
  --volume=/var/lib/docker/:/var/lib/docker:ro \
  --publish=8080:8080 \
  --detach=true \
  --name=cadvisor \
  google/cadvisor:latest
```

A bit frustatring as we can see nothing wrong seems to be going on. CPU usage is fine and memory is under80%.
![_config.yml]({{ site.baseurl }}/images/CADVISOR-SCREEN1.png)
![_config.yml]({{ site.baseurl }}/images/CADVISOR-SCREEN2.png)

Fortunately  Maven can be optimized to run the build in paralell among several CPUs   

# CI build
Instead of installing Jenkins on a machine and then add more tools we can use the same approach so changes in the repository trigger a build which in fact launches

# A note in multi-module projects

Our project has the following structure were <Parent> acts as module aggregator

> /path/to/projectroot/Project A
	> childProjectA1
	> childProjectA2
> /path/to/projectroot/Project B
> /path/to/projectroot/PARENT

So our final command looks like

```
docker run -it --rm -w /opt/maven -v $PWD:/opt/maven -v $HOME/.m2:/root/.m2 maven:3.3-jdk-8 mvn clean install -f ./PARENT/pom.xml
```
