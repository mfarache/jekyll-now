---
layout: post
title: Building a CD pipeline with docker and jenkins
---

My objective is prototype  a CD flow with several stages including manual steps before the artifacts are released and deployed to each stage.  In a continuos deployment flow there are several points where QA may need to perform several regression testing or manual tests before signing off a release.

We would like to release our code as final images instead of artifacts that are pushed to the environments before a new image is built.

## Why?
Our current release can be improved significantly. The current process involves pulling artifacts from a nexus repository.

The artifacts are retrieved using curl from the target machine to populate a workspace used by docker to build an image.
```
SCM > ARTIFACT REPOSITORY >  ENVIRONMENT > DOCKER BUILD IMAGE  > DOCKER run
```

The inefficiencies are obvious
First each environment has to retrieve the artifacts to build an image on each environment. There are no guarantees that the artifacts retrieved are identical and therefore the images could be different.
The image is rebuild from the scratch and is very time consuming.
At the same time the process cannot be used by our developers who setup their environment in different ways.

With the new approach the process will be

```
SCM > DOCKERHUB  REGISTRY >  ENVIRONMENT > DOCKER PULL IMAGE  > DOCKER run
```

Once images are in the repository every developer should be able to run exactly the same version as in production via docker compose (as we have  about 9 containers)

![_config.yml]({{ site.baseurl }}/images/DockerWorkflow.png)


The new build process should have at least the following stages to prove the typical scenario
- Checkout code
- Build code artifacts
- Unit and integration tests
- Build images & store in registry
- Manual checkpoints
- Deployments to stage.

The deployment step probably would imply using SSH agentsy to launch a script remotely. The script will just run a docker pull to retrieve the image.

Other parameters can be added as tag, version,etc so the registry keeps a good traceability of the images.

Images should rely on external configuration, so the same images can be reused 100% across environments. It can be easily achieved just mapping volumes from the container configuration directory to the host that would keep the configuration settings.

## The proof of concept
The initial requirements in mind where that solution should involve using Docker as way to containerize Jenkins and include building images as part of the build cycle.

After some research I found out the existence of several Jenkins plugins like Docker and Docker slaves that were interesting.

They could be the base of another post about how to use several Jenkins slaves that could run our jobs using embedded containers.
Each embedded container will be responsible only of building our artifact , keeping clean our jenkins master instance.
There are well known images like mvn3.3.3-jdk-1.8 which incorporates a full containerized JAVA build environment following the new [builder pattern paradigm][1]  

For the purpose of the test I chose the pipeline plugin available with Jenkins 2.0

# Pipeline definition

So lets setup or pipeline using Jenkins

First approach would be using Docker in docker (Dind), but after some reading I found that there are a few "gotchas", so we can achieve the same result just by mounting the docker.sock file. It looks like Docker-in-Docker, feels like Docker-in-Docker, but it’s not Docker-in-Docker: when this container will create more containers, those containers will be created in the top-level Docker. You will not experience nesting side effects, and the build cache will be shared across multiple invocations.

```
docker run -d -p 8080:8080 -p 50000:50000 --name jenkins_server --net=continuus-deployment-network -v /var/run/docker.sock:/var/run/docker.sock -v /tmp:/var/jenkins_home jenkins_server
```

So you can visit http://localhost:8080 and complete the steps in order ti finish your Jenkins configuration.  The container is sharing the main jenkins folder with its host , so even if you stop and restart the container your job settings, plugins,etc will persist.


Be sure the Jenkins pipeline plugin is installed which lets you orchestrate automation, simple or complex. See [Pipeline as Code with Jenkins][2] for more details. As a heads up you can define your pipeline with groovy DSL. Pipelines are executed in nodes. Within each node you can define a set of stage made of steps. For simplicity everything will be run on a single node, but is even possible to configure docker nodes to perform specific steps.

Follow the creation of a new job chosing  Pipeline option

I choosed to run a parameterized build so I can reduce build time chosing if we skip test or if we use docker. Not using docker would streamline the pipeline in case we want to skip the steps of building & pushing images.

+ Definition: Pipeline from SCM
+ Chose your favorite SCM settings where you store the code you want to build & release
+ Script Path: Jenkinsfile

Note: Remember the Jenkinsfile must be in the root of your repository!

My Jenkinsfile contents are:

```javascript
#!/usr/bin/env groovy

import hudson.model.*
import hudson.EnvVars
import groovy.json.JsonSlurperClassic
import groovy.json.JsonBuilder
import groovy.json.JsonOutput
import java.net.URL

properties([[$class: 'ParametersDefinitionProperty', parameterDefinitions: [
[$class: 'BooleanParameterDefinition', name: 'skipTests', defaultValue: false],
[$class: 'BooleanParameterDefinition', name: 'skipDocker', defaultValue: false]
]]])

try {
node {

def workspace = pwd()+"@script"
def registryUser ="<YOUR_DOCKER_USERNAME>"
def registryPassword ="<YOUR_PASSWORD>"

def image = "mock-palm"

stage 'Download Code'  

echo "\u2600 BUILD_URL=${env.BUILD_URL}"

echo "\u2600 workspace=${workspace}"

stage 'Build'
echo "Building"
def mvnHome = tool 'M3'
sh "cd ${workspace} && ${mvnHome}/bin/mvn clean install -DskipTests"
stage 'Unit Test'
echo "Unit testing"
sh "cd ${workspace} && ${mvnHome}/bin/mvn surefire:test"

stage 'Integration Test'
echo "Integration testing"

stage 'Build Image'
echo "Docker Build"
if (Boolean.valueOf(skipDocker)) {
      echo "Docker build Skipped"
} else {
	  sh "cd ${workspace} &&  docker build -t ${image} ."
}

stage 'Push Image Registry'
if (Boolean.valueOf(skipDocker)) {
      echo "Skipped"
} else {
   echo "Login as ${registryUser} against registry DockerHub"
   sh "docker login -u ${registryUser} -p ${registryPassword}"
   echo "Login sucessful"
   sh "docker tag ${image} ${registryUser}/${image}"
   echo "Tagging ${image} within ${registryUser}/${image}"
   sh "docker push ${registryUser}/${image}"
   echo "Pushed ${registryUser}/${image}"
}

stage 'Deploy to DEV'
echo "Deploy to DEV"

stage 'Deploy to QA'
echo "Deploy to QA"
input 'Do you approve deployment to QA?'

stage 'Deploy to PRD'
echo "Deploy to PRD"
input 'Do you approve deployment to QA?'

} // node
} // try end
catch (exc) {
/*
  Your error handling should notify via slack or email
*/
}

```
This is how my pipeline looks like after a succesful build

![_config.yml]({{ site.baseurl }}/images/JenkinsPipeline.png)

Other funny facts I learnt while doing this proof of concept

## Out-of-Sync clock in MAC
Anytime I left my MAC on hibernate I realized that the dates of the container where left "frozen in the past".
So If I close the lid during 8h, my container will not realize and will think is in the past.
This has an annoying effect when running a jenkins within docker... You will be wondering why..
Well everytime you checkout code, jenkins thinks the SCM server time is wrong and do not pick up recent changes.

As a workaround I run this immediately before running a job to sync clocks. Based on the fact that docker is not purely native.
Docker on Mac runs under a VM with alpine.
```
docker run --rm --privileged alpine hwclock -s         
```

# Useful links

+ [builder pattern paradigm][1]  
+ [Pipeline as Code with Jenkins][2]
+ [Docker in docker using Jenkins slaves][3]
+ [Why not use docker in docker][4]
+ [Building images with Docker using jenkins pipeline][5]

[1]: http://blog.terranillius.com/post/docker_builder_pattern/
[2]: https://jenkins.io/solutions/pipeline/
[3]: https://github.com/tehranian/dind-jenkins-slave
[4]: https://jpetazzo.github.io/2015/09/03/do-not-use-docker-in-docker-for-ci/
[5]: https://blog.nimbleci.com/2016/08/31/how-to-build-docker-images-automatically-with-jenkins-pipeline/
