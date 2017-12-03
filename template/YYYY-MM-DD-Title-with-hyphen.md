---
layout: post
title: Deep dive on amazon lambdas
tags: [ AWS, AWS Lambdas, S3, Jenkins, docker]
---

The following post will attempt to go through best practices, networking and things to bear in
mind while developing with AWS lambdas

# Best practices

+

# Networking



## 1. Install the CloudFormation plugin in your jenkins server.

## 2. Define a cloudFormation template.

There are many snippets at [CloudFormation templates][3]
I created a new S3 bucket to organize out templates. We uploaded it to S3 so later we can refer to it just using its S3 URL.

![_config.yml]({{ site.baseurl }}/images/s3-cloudformation-template.png)


# Useful links

+ [CloudFormation jenkins plugin][1]

[1]: https://wiki.jenkins.io/display/JENKINS/AWS+Cloudformation+Plugin
