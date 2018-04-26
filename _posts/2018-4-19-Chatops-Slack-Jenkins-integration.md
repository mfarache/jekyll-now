---
layout: post
title: Jenkins Pipeline integration with Slack 
tags: [ Jenkins, Slack, CI, chatops ]
---

The following post describes a step by step guide to include ChatOps in your project using Jenkins and Slack.

# Benefits

Our jenkins pipeline has a Post-Build notification step that inform us about the status of our last build. It's fair to say that we have already automated our Continuos Integration process.
...however is easy that these emails tend to be ignored or they fall in a folder via some email filtering rule. 

Our company uses Slack as a way to share information, provide discussion chat rooms per customer and project,etc. A natural step ahead is improving our Continuous Integration pipeline with Slack so everyone in the development team is aware when something went wrong. 

This requires also some discipline from the developent team. Ignore the channel alerts is not a good idea, we would fall back with the same issues we had with emails.
You break it, you fix it! 

We can see a few benefits moving to online notifications approach:

+ Improved collaboration
+ Increased visibility and awareness
+ Ability to react build failures faster
+ Awareness of deployments, when happened and why
+ Reduction in emails

Let's see how easy is to setup from the scratch. It took me less than 30 minutes to configure everything. 

# Create a channel to receive Jenkins notifications

We need to create a new Slack channel to receive notifications from Jenkins.
Once we have created the channel  we need to configure it:
Select your channel and then you can find a configuration option close to the search text box.
Select Add an App. From the following screen select Jenkins.

![_config.yml]({{ site.baseurl }}/images/SLACK_CI_SLACK_ADD_APP.png)

Now we just need to add a configuration to indicate which channel Jenkins should post to.

![_config.yml]({{ site.baseurl }}/images/SLACK_CI_SLACK_ADD_CHANNEL.png)

Once the integration is added a new page will guide through you the remaining configuration steps that need to be done via Jenkins. Keep that page open, as you will need it handy for the next step.


# Enable slack integration in Jenkins

Slack integration requires installation of the plugin Slack Notification plugins.
From the home page Go to Manage / Manage Plugins
Search for Slack and install it. 
Restart is not necessary so we can proceed to configure our plugin

From the home page Go to Manage / Manage Plugins

We will find two Slack sections. 
We can skip the first section as it's intended for webhook configurations. This is the section that we would use if we would like to configure for example git actions.

However we are focusing on this post only on Build status notifications.
You need to fill the following form with the correct entries.

![_config.yml]({{ site.baseurl }}/images/SLACK_CI_JENKINS_CONFIG.png)

Adding the BaseUrl, the token and the channel is enough. 

# Add notification to your job

Once everything is setup we need to configure our job to send notifications.
The easiest way is adding a Post-Build action and filter the type of events we are interested for the notifications

In our case we are using the Pipeline plugin so we would do it modifying our pipeline DSL. 
If you are not familiar with it, I wrote a post about how to use it [here][1]

Be sure the Jenkins pipeline plugin is installed which lets you orchestrate automation, simple or complex. 

See [Pipeline as Code with Jenkins][2] for more details. As a heads up you can define your pipeline with groovy DSL. Pipelines are executed in nodes. Within each node you can define a set of stage made of steps. For simplicity everything will be run on a single node, but is even possible to configure docker nodes to perform specific steps.

I will omit some details and work with a hyper-simplified pipeline just to show the idea
A simple pipeline with 3 steps:
+ checkouts code from git (replace GIT_REPO and credentialsId)
+ compile the code
+ run unit test

```groovy
try {
    node {
        stage ('Download Code') {
            echo "Tag selected: ${gitTAG}"

            def GIT_REPO='YOUR_URL_REPO_HERE'

            echo "Downloading code from: ${GIT_REPO}"
            
            checkout([$class: 'GitSCM',
                branches: [[name: gitTAG]],
                doGenerateSubmoduleConfigurations: false,
                extensions: [[$class: 'CleanCheckout']],
                submoduleCfg: [],
                userRemoteConfigs: [[credentialsId: 'YOUR_CREDENTIALS_ID', url: GIT_REPO]]
            ])

            echo "\u2600 BUILD_URL=${env.BUILD_URL}"

            echo "\u2600 workspace=${workspace}"
        }

        def dockerTag='development'

        stage ('Build') {
            if (Boolean.valueOf(skipBuild)) {
                echo "Build is skipped"
            } else {
                echo "Building"
                def mvnHome = tool 'maven-3.3.9'
                sh "cd ${workspace} && ${mvnHome}/bin/mvn clean install -DskipTests -Dbuild.number=${BUILD_NUMBER}"
            }
        }

        stage ('Unit Test') {
            if (Boolean.valueOf(skipTests)) {
                echo "Integration tests were skipped"
            } else {
                echo "Unit testing"
                def mvnHome = tool 'maven-3.3.9'
                sh "cd ${workspace} && ${mvnHome}/bin/mvn surefire:test"
            }
        }
    } // node
    notifyBuild('SUCCESSFUL')
} // try end
catch (exc) {
   notifyBuild('ERROR')
}
```

The following function sends a message to a slack channel with status and build metadata. This method needs to be invoked from your DSL node definition



```groovy
def notifyBuild(String buildStatus = 'STARTED') {
  // build status of null means successful
  buildStatus =  buildStatus ?: 'SUCCESSFUL'

  // Default values
  def colorName = 'RED'
  def colorCode = '#FF0000'
  def subject = "${buildStatus}: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'"
  def summary = "${subject} (${env.BUILD_URL})"
  def details = """<p>STARTED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]':</p>
    <p>Check console output at &QUOT;<a href='${env.BUILD_URL}'>${env.JOB_NAME} [${env.BUILD_NUMBER}]</a>&QUOT;</p>"""

  // Override default values based on build status
  if (buildStatus == 'STARTED') {
    color = 'YELLOW'
    colorCode = '#FFFF00'
  } else if (buildStatus == 'SUCCESSFUL') {
    color = 'GREEN'
    colorCode = '#00FF00'
  } else {
    color = 'RED'
    colorCode = '#FF0000'
  }

  // Send notifications
  slackSend (color: colorCode, message: summary)
}
```

After triggering a build we receive a notification and we can see the status

![_config.yml]({{ site.baseurl }}/images/SLACK_CI_RESULT.png)

# Useful links

+ [Building a CD pipeline with docker and jenkins][1]
+ [Pipeline as Code with Jenkins][2]
+ [Pipeline slackSend syntax][3]
+ [Plugin code][4]

[1]: https://mfarache.github.io/mfarache/Continous-deployment-pipeline-jenkins-docker/ 
[2]: https://jenkins.io/solutions/pipeline/
[3]: https://jenkins.io/doc/pipeline/steps/slack/
[4]: https://github.com/jenkinsci/slack-plugin

