---
layout: post
title: Log aggregation alternatives using docker - Using  papertrail
tags:   [ logging, papertrail, docker ]
---

I will explore in a series of post how Docker can be used to detect proactively errors via logging monitoring. First I will use Papertrail as a SaaS solution, but the space is crowded of multiple alternatives to be explored.

# Using Papertrail

Moving to the cloud seems a natural fit for using this SaaS approach.
You can easily tail the logs from their web UI very fast, you can even stream its events online with nearly no delay.
There are several options / plans available so depending on the amount of logs can be a feasible option.
Open an account in PaperTrails is straight forward. After that the only thing you need to do is create a Log Destination.
Once you have done that you are ready to go.

```
docker run --name="logspout" --volume=/var/run/docker.sock:/var/run/docker.sock gliderlabs/logspout syslog+tls://logs5.papertrailapp.com:35006
```

With that simple command , any new Docker container you spin up will spits its output to Papertrail.
Papertrail allows you to search, create alerts from the search, send notifications and integrate with a OpsChat approach using Slack for example among others. You can create different channels and classify your errors by team or application or severity and get everything automated in a snap. Using webhooks you could even create automatically JIRA tickets if you feel lazy!

 ![_config.yml]({{ site.baseurl }}/images/PAPERTRAIL.png)

You can even bookmark a specific line within the log file and share it with your team so everyone can see what is happening.
The online web panel allows to do seek positions by date, patterns, enable/disable live tails so there are no more excuses to ignore errors in your local environment and why not in a production environment if your company has some budget to spent on decent SaaS logging.

Papertrail allows you to create logical groups of container to make monitoring easier.

It´s worhty to explore options like labeling containers or creating specific logging destionations
For example you can label some container with a tag <APPSERVER> and then run your logsput container to restrict sending information from the containers labelled <APPSERVER>

There are plenty options, visit https://github.com/gliderlabs/logspout for further information.

Other goodies you can find useful is the ability to filter and ignore completely incoming messages via regex matching or simple string matching. It occurs to me you may not be interested in DEBUG messages or you want to restrict the kind of information,

# How can we handle multiple logs?

I was pretty satisfied with the approach till I face a little stopper.
Sometimes our application may write different logs and the default log outpout provided by docker is not enough.
In this case papertrail does not helps too much.
As an example our integration layer runs a ESB Mule on docker and we have specific logs per flow deployed on our Mule.

What can we do?

We could run our container with a volume

```
docker run -d -p 8001:8001 -v <PATH_TO_MY_LOGS_IN_HOST>:<PATH_TO_MY_LOGS_IN_CONTAINER> --name mule37 my-private-mule-image
```
And then start this awesome container with the same
```
docker run -d -v <PATH_TO_MY_LOGS_IN_HOST>:/host/logs janeczku/remote_syslog2 -d logs.papertrailapp.com -p 35006 --tls=true /host/logs/*.log
```
Voila, your logs have been added automatically to Papertrail.

It would be great if I could do that online streaming log output... but only could find batch processing of logs.

# Other logging alternatives?

I´m just scratching the surface here as there are many logging servers / solutions out there, just to name a frameworks

+ Splunk (Commercial and expensive license enterprise model)
+ Logstash ( we will see in the next post how we can set a local ELK solution)
+ Fluentd
+ Loggly
+ Logentries
+ Graylog ..

So the tools are right there, we just need to get to use them !!

# Final thoughts

Each member of our development team can have his own logging search tool on his own laptop that facilitates searching for issues and troubleshooting. When you run a big monolithic application the usual suspects vi,grep,cut, sed do the job but once you break down your components in micro services or just smaller application with clear responsabilities,  logs start to crop in and finding where the error may be happening may get tricky.

Other advantage is that we can be proactive instead of reactive. We can configure alerts on specific text patterns and be notified when instances of this error happen. We have experienced that many errors are hidden in the logs and that no one payed attention just because no one even noticed them.
