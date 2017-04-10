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
+ Developers may switch projects easily without installing tools or frameworks that contribute to reduce laptop performance over the time.
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

Then I realized at the settings of my Docker Preferences that only 2GB were allocated for the containers!
I increased that up to 10GB and that made the difference. That and also using the 4 cores and some mvn flag to work offline.
There might be even further improvements if we play with the JVM settings and MAVEN_OPTS, but at least I managed to run the whole process nearly as fast as it would be running natively on my laptop.

More research on this recommends using volumes so I created one, and out of curiosity you can see where Docker stores the data
```
------------------------------------------------------------
~/Documents/workspace » docker volume create my-maven-volume                                                                                                 mfarache@OEL0043
------------------------------------------------------------
~/Documents/workspace » docker volume inspect my-maven-volume                                                                                                mfarache@OEL0043
[
    {
        "Driver": "local",
        "Labels": {},
        "Mountpoint": "/var/lib/docker/volumes/my-maven-volume/_data",
        "Name": "my-maven-volume",
        "Options": {},
        "Scope": "local"
    }
]
```

So now lets use the volume
```
docker run -it --rm -w /opt/maven -v $PWD:/opt/maven -v my-maven-volume:/root/.m2 maven:3.3-jdk-8 mvn clean install -f ./discovery-parent/pom.xml -DargLine="-XX:+TieredCompilation -XX:TieredStopAtLevel=1" -T 1C
```
Once all the internet has been downloaded to your volume, it can be shared across all your development team and adding the offline flag your build will run much faster. However is fair to say that speed cannot be compared with a native build so we cannot see this approach as an improvement in productivity. Our Arquillian tests are time consumings and when run on a container you can go away and take not only one cup of coffee. Its soo slow!

Lets assume for a moment that our project builds very fast and the difference with native build is less than 10%.
Once our build ends there are multiple path alternatives we can take:

+ The first one would be build a image straight away and push to our own registry. I discussed several alternatives in a previous post
[Using private docker registry alternatives][1]

[1]: https://mfarache.github.io/mfarache/Using-private-docker-registry-alternatives/

The drawback of the approach is that our image will have all the toolset (mvn itselg, the .m2 repository which was mapped, and the code downloaded from our SCM which was also mapped. The image size will be huge.
So not a good idea when all we need is a bunch of jar files. What can we do?

+ We could use copy command (docker cp) in order to extract the jar files from our container. The artifacts would be the basis of a workspace where we would have a Dockerfile with simple instructions to run our java code. Copying manually files from docker containers seems a bit cumbersome so using volume sharing we could streamline significatively the process. The Builder pattern for Docker can be summarized combining 2 different Dockerfiles

++ The first one builds the code so it require a toolset and libraries to do so. The outcome is that we have an artifact ready
++ The second dockerfile just takes the result of the first build process , extends from a basic image , adding the binary so the image result is considerably smaller.

Recently a [Docker PR][2] has just been merged to enable multi-stage builds. Lets see with an example how this will affect our build process. We will be able to do everything in a single Dockerfile:
[2]: https://github.com/docker/docker/pull/32063

```
FROM maven:3.3-jdk-8 as builder
WORKDIR <PATH_TO_YOUR_CODE>

RUN mvn clean install -T4
#This will generate the jar file in <PATH_TO_MY_JARFILE>

FROM jdk-8:latest  

COPY --from=builder <PATH_TO_MY_JARFILE>   .
CMD ["java -jar <JARFILE>"]
```

#Final thoughts

Depending on the size of the project the previous approach could be valid, but in our case we will stick to local builds non-dockerized as one of the main drivers of commit soon-commit often goes against long lasting builds.
