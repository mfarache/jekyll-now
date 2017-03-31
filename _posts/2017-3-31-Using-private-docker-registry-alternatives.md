---
layout: post
title: Using private docker registry alternatives!
---
Running containers within containers and using private registries implies some configuration issues. Several options were tried including
+ insecure registries
+ registry proxied by nginx to setup TLS negotiation
+ registry images using certs and CA crts.

I will explore the alternatives available and share some instructions that I hope will be useful to someone who faces the same dilemma.

## 1. Run a private docker registry as docker container

### 1.1 Running a registry (non-secure)
```bash
docker run -d -p 5000:5000 --restart=always --name registry registry:2
```
You need to add the --insecure-registry flag /etc/default/docker.

In a MAC docker runs on a internal VM running alpine accesible using

```bash
screen ~/Library/Containers/com.docker.docker/Data/com.docker.driver.amd64-linux/tty
ctr+a+d EXIT
```

NOTE: Insecure registries does not support login/password access

But this approach will not allow remote push from the container :(

### 1.2 Running a private registry

#### 1.2.1 Using SSL certs accesible from the docker registry

First we need to generate self-signed certs
```bash
docker run --rm -v /tmp/registry/ssl:/certs -e SSL_DNS=registry paulczar/omgwtfssl
```
The host directory where the certs are  will be mapped as a volum with the registry once container is started, so registry pick up the SSL config

The registry.env file should look like

```
Contents of the file
# location of registry data
REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY=/tmp/registry/data

# location of TLK key/cert
REGISTRY_HTTP_TLS_KEY=/tmp/registry/ssl/key.pem
REGISTRY_HTTP_TLS_CERTIFICATE=/tmp/registry/ssl/cert.pem

# location of CA of trusted clients
REGISTRY_HTTP_TLS_CLIENTCAS_0=/tmp/registry/ssl/ca.pem
```

Inspiration comes after finnding this post http://tech.paulcz.net/2016/01/deploying-a-secure-docker-registry/

```bash
docker run -d --name registry -v /tmp/registry:/opt/registry -p 5000:5000 --restart always --env-file /tmp/registry/config/registry.env registry:2
```

#### 1.2.2 Runnning a secured docker Registry with nginxproxy

You need to copy cert files, created in the previous steps
```bash
/tmp/registry/ssl » mkdir -p /tmp/security/external
/tmp/registry/ssl » cp cert.pem /tmp/security/external
/tmp/registry/ssl » cp key.pem /tmp/security/external

```

We will be protecting our registry with simple http auth lets create a htpasswd. That could be done using some fancy docker image but this time I went through the simplest path

Visit http://www.htaccesstools.com/htpasswd-generator/

Assuming you use registry:XXXXXXX
copy the results so your file looks like

```
/tmp/security/external » cat docker-registry.htpasswd
registry:$apr1$Rtbdcp0T$M/mH/oUjyBNRsrOSTrQUz.
```

So now we start a registry and a nginx proxy to provide upstream TLS negotiation
```bash
docker run -d --name registry -v /tmp/registry:/registry -e "SETTINGS_FLAVOR=local" -e "STORAGE_PATH=/registry" registry
```

```bash
docker run -d -p 443:443 -v /tmp/security/external:/etc/nginx/external --link registry:registry --name nginx-registry-proxy marvambass/nginx-registry-proxy
```
Once our containers are up and running

```bash
/tmp/security/external » docker login https://localhost:443       
Username: registry
Password:
Login Succeeded
```

Wohoo!!!! so now we can interact with it pushing/pulling images in a secure way.

## 2. Using Nexus 3

Nexus acts as artifact repository and is part of our CI cycle. The latest version incorporates a feature that enables
a docker registry with no much hassle.
```
docker run -d -p 8081:8081 -p 8082:8082 -p 8083:8083 --name nexus sonatype/nexus3
```

Access via browser to localhost:80801 and create a repository. name it docker. We have 2 options
+ exposing  http(8082)
Docker Daemon can stand up instances with the --insecure-registry flag to skip validation of a self-signed certificate. But the repository manager does not support the use of the flag, as it generates known bugs and other implementation issues.

+ exposing https(8083)
The option to secure Nexus 3, did not work. The port was not even listening.
```
docker logs -f nexus
```
throws a hellish stack trace, which seem to indicate that the embedded Jetty needs to be configured internally using SSL.
There is an excellent article [Securing Jetty via SSL][7] but I did not think was worthy to spend more time, because that outcome would not differ from previous scenarios as eventually I would face the same issue running under another container.

At this point, the Docker Registry is up and running, but you can’t access it from a docker client because Docker requires the registry to run on SSL.

## 3. Using a private DockerHub repository
Create an account in DockerHub, i.e dockermau using your email.

To see that everything works
```bash
~ » docker login --username=dockermau                                                                                     
Password:
Login Succeeded

```
Now we are ready to pick one of our local images (busybox, if you do not have it you should do docker pull busybox first)

```bash
docker tag busybox dockermau/busybux
docker push dockermau/busybux
The push refers to a repository [docker.io/dockermau/busybux]
c0de73ac9968: Mounted from library/busybox
Pushin to remote DockerHub registry
latest: digest: sha256:32f093055929dbc23dec4d03e09dfe971f5973a9ca5cf059cbfb644c206aa83f size: 527
```

This solution works fine, the docker hub repository can be marked as private so only repository owner can pull/push images
To improve speed and reduce times, we can add a Squid cache container that would improve performance caching images near end

## 4. Using Amazon ECR
There is a great aritcle [here][1]
Luckily enough was happy with option 2, so there was not reason to get tied to AWS again. Dockerhub does the job pretty well.

# Gotchas

While using a private registry works fine pushing from the host to the container, the problems arise where you want to push images from another container.

Under MAC the main issue found was that was possible to push images from the host
```
push localhost:5000/<myimage>
```
However this was not possible from the container itself
```
push registry:5000/<myimage>
```
After one day banging my head against the wall , trying every possible alternative or workaround I gave up and went for the simplest solution
based on hosting images in private repositories in DockerHub.

Really the purpose of my proof of concept is not learning the inner gritty details of security around docker containers, instead try to set the ground to prepare a continuos development pipeline using Jenkins. which will be the base of my next bog entry.

#Related links
+ [Using Amazon ECR as Docker registry][1]
+ [A nginx registry proxy][2]
+ [Using Nexus 3 as Docker registry][3]
+ [Nexus 3 image][4]
+ [Securing Jetty via SSL][5]
[1]: http://rancher.com/using-amazon-container-registry-service/
[2]: https://hub.docker.com/r/marvambass/nginx-registry-proxy/
[3]: http://www.sonatype.org/nexus/2016/06/29/using-nexus-3-as-a-private-docker-registry/
[4]: https://github.com/sonatype/docker-nexus3
[5]: http://www.eclipse.org/jetty/documentation/current/configuring-ssl.html
