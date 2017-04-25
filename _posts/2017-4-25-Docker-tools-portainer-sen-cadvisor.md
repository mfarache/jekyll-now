---
layout: post
title: Docker toolset to inspect your containers: Sen, Portainer, caDvisor
tags:   [ cadvisor, sen, portainer, docker]
---

A very short post showing some tools that can help you managing your docker containers in a intuitive way. We will see a terminal management tool, a nice web UI Portainer and a monitoring tool: caAdvisor from Google.


# Google caAdvisor

cAdvisor allows you getting stats information about resource usage and the various performance metrics for your containers. You have a simple web UI that can be accessing hitting
http://localhost:8080 ( in case you mapped 8080 to your host port as I did in the example below)

```bash
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

Some screenshots to get an idea

![_config.yml]({{ site.baseurl }}/images/CADVISOR-SCREEN1.png)
![_config.yml]({{ site.baseurl }}/images/CADVISOR-SCREEN2.png)

It also exposes an API that gives you access to valuable information such as

+ List of subcontainers
+ ContainerSpec which describes the resource isolation enabled in the container
+ Detailed resource usage statistics of the container for the last N seconds (N is globally configurable in cAdvisor)
+ Histogram of resource usage from the creation of the container


# Useful links

+ [SEN Term console][1]
+ [Google cadvisor][2]
+ [Portainer][3]

[1]: https://github.com/TomasTomecek/sen
[2]: https://github.com/google/cadvisor
[3]: https://github.com/portainer/portainer
