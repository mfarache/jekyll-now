---
layout: post
title: Thoughts on using SQS with lambdas
tags: [ aws, sqs, lambda ]
---

Some random thoughts on which are the options available to integrate SQS queue with lambda processing.

# Background

In previous entry I explained that my next proyect  would use of Amazon serverless technologies like S3, SNS, SQS, Step Functions and Lambdas via an API Gateway.  

As I'm not familiar with them, I summarize here what I learnt re SQS and lambdas, in case it can be of any help for anyone ( probably me!) in the future.

# SNS And SQS

+ Typical use case is the definition of a SNS topic to push messages and a SQS suscribed to the topic. Then SQS can be configured to trigger a lambda on arrival of each message.
The first reasonable doubt is why do not do the same with SNS, ie linking directly SNS with a lambda. The main reason is that using SQS gives us a temporary storage with retries, and guarantee at least one time delivery. SNS does not offer that guarantee.

# Choosing a SQS type

+ SQS guarantess one time delivery.
+ There are two different types
    + SQS standard queue
    + SQS FIFO guarantees read exactly once.

+ During a long time, if you wanted to trigger a lambda on arrival of your message the only option left was using the Standard one. However during AWS:reinvent 2019 was announced that now that feature is also supported by SQS Fifo queues.

+ SQS FIFO is not compatible with SNS yet. So anyone intending to trigger lambdas should use the combination SNS + SQS Standard + Lambda.

# SQS message processing 

+ Filtering policies can be definied on SNS side when a SQS is subscried to a topic. It's useful if we configure SQS fan-out, ie many SQS subscribed to the same topic. Valid scenarios where we can use this approach different types of messages are placed in the SNS topic, but you need to add code in your lambdas to filter and skip those which you are not interested. Instead of adding filtering logic to your lambda you define a filtering policy. The approach improves the performance due to the reduction of lambda invocations due to arrival to the SQS queue.

+ Whenever a message is consumed, SQS will not remove it from the queue. The consumer  needs to send a separate request in order to delete the message (meaning that message is already processed it succesfully). If we forget to remove it, other consumers could pick the message again once the visibility period expires. This concept also called visibility timeout, is a period of time during which Amazon SQS prevents other consumers from receiving and processing the message.

+ In case we do not have idempotent processing of messages we always need to remove messages, otherwise we could incur on inconsistencies due to processing the message twice or more times.

+ An alternative approach to lambdas triggered on arrival of messages to SQS, is to implement polling. Polling can be implemented using EC2 consuming messages but that would more expensive due to the cost related with the EC2 being up and running forever. 

+ Another alternative we could also be using would be configuring a CloudWatch Event rule triggered every minute or 10 minutess. The rule will trigger lambda will receive the messages, but this will end up having the same potential issues described in the section related to concurrency.

+ Other people has explored using a recurrent lambda function that tries to consume as much messages as possible , stay idle for a while and launch another executions recursively. The cost really of this approach is not much (below 5$ per month ).

+ If Amazon SQS queue and Amazon SNS topic are in different AWS accounts, the owner of the topic must first confirm the subscription.

+ SQS allow us to configure DLQs for messages received N times but not processed successfully yet. Ideally we could reprocess them with CloudWatch alarms, so maybe transfer back from the DLQ into the original SQS.

# SQS and lambda concurrency 

+ The concurrency limit determines how many function invocations can run simultaneously in one region. In case the concurrency limit is too low, Lambda can throttle due to taking too many messages off a busy queue.

+ The real issue with low concurrency values (<5) is that internally AWS provide us with a Lambda-SQS long-poll service with 5 connections polling the queue. Sad news is that we cannot control that number of services running with any parameter.
If our concurrency is set to one, the first connection will consume the batch size of 10 messages, however the other 4 connections will fail as we have set up 1 to be the concurrency limit.
The other 4 connections as consequence of our setup will be throttled and the messages are returned to the queue.
+ Each message has metadata which indicate the times the message has been received. Every time the message is returned that counter is increased. Once the counter reachs the threshold value configured for the queue, the message will be moved to a DLQ in case you have a redrive policy. We lost our message! 
+ There are some recommendations from AWS saying that the risk can be mitigated if the visibility time out period is configured to be x 6 times, the timeout of your lambda function. 
