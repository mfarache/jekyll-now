---
layout: post
title: Storage options S3 vs DynamoDB
tags: [ aws, s3, storage, dynamodb, ]
---

The next proyect I will onboard will make heavy use of serverless technologies like S3, SNS, SQS, Step Functions and Lambdas via an API Gateway.  The project as any other requires store and retrieve elements as part of processing the events.

I will collect some thoughts around the factors that one could consider in order to chose a valid storage option fitting our requirements.

# The contenders: S3 vs DynamoDB

A reasonable dude one may raise is whether we required a full blown NoSQL database or we could live just using S3 Storage system.

Our contenders will be S3 and DynamoDB.

I have organized in several subtopics exposing pros and cons together with some considerations.

## Data consistency

When we deal with distributed nodes, consistency means that the data returned from a request should be always the same regardless which node from the cluster is picked.

Let's see how S3 handles eventual consistence and which are some side-effects.

First of all is worthy clarifying that S3 buckets provide strong read after create consistency (new objects). However the eventual consistency happens when updating an objet or deleting it.

An an example, if a process writes to an existing bucket on S3, and immediately afterwards other process reads from it,that read may or may not include the most recent up-to-date data.

Other tricky situations steem from the fact that operations are cached by key. Why does it matter?  

Imagine the pattern create object if does not exist. During a period of time, it would mislead the client returning not existing data when really data really exists! In summary you could get two different versions of the same object for a period of time.

Therefore if you need read-after-update consistency you need to consider DynamoDB as a serious contestant to avoid corruption caused by reading stale data.

With DynamoDB you can optionally enforce strong read consistency (By default is eventually consistent reads). This means that DynamoDB is better if you need to ensure that two different processes always get exactly the same information while a record is being updated.

## Latency / Throughput

Although S3 is a super-reliable system in terms of not losing your data, I'm afraid we cannot say the same about latency. You can expect times below 400ms, however there is no guarantee that on certain conditions that time can be well over 4secs. If your application cannot cope in that scenario,

S3 is designed for throughput, not necessarily predictable (or very low) latency.
In fact its latency is much higher than a real database (be it a sql database, dynamodb, elasticsearch, etc)

S3 tends to lend itself better to data that doesn't have as much concurrency issues, or data that is mostly static.

There are some tips on performance that can reduce significantly s3 latency

+ Latency on S3 operations depends on key names since, prefix similarities become a bottleneck at more than about 100 requests per second.

+ Using smart use of object prefixes in an Amazon S3 bucket would imply that reads can be parallelized.

+ Some times you do not need to access the full content of the object in S3. Imagine files where you are aware of its structure, and you are only interested on some metadata located in the header of the file. S3 gives you the ability to send range gets whereby you only get the number of bytes specified on the range.

+ In case your requirements are low, low latency but still happy accessing as a key value AWS provides straigt forward integration with ElastiCacheRedis so your application could use lazy load to populate it. An alternative is populating the cache as far as the elements are created in S3 triggering a lambda.

On the other side DynamoDB is designed for low latency and sustained usage patterns. If the average item is relatively small, especially if items are less than 4KB, DynamoDB is significantly faster than S3 for individual operations.

DynamoDB Global Tables supports multi-master replication, so clients can write into the same table or even the same item from multiple regions at the same time, with local access latency.

## Atomicity and Transactions

With S3 atomic batch operations on groups of objects are not possible.
S3 operations generally work on entire items and it’s difficult to work with parts of an individual object.

There are some exceptions to this, such as retrieving byte ranges from an object.

Obviously if you have strong requirements on transactionality DynamoDb
is a clear winner. It also enables multiple writers modifying
concurrently  properties of the same item, or even append to the same array.

DynamoDB can efficiently handle batch operations and conditional updates, even atomic transactions on multiple items.

## Scaling

Although DynamoDB can scale on demand, it does not do that as quickly as S3
There are ways to improve latency using supports secondary indexes, arbitrary queries, and triggers (through Lambda)
Dynamo does not perform well when does not use indexes and is forced to use scans.

## Reporting

Often reporting and business intelligence requirements are neglected.

Imagine you have built your application and using S3 seemed to fit perfectly based on your latency, consistency and scaling requirements.

Suddenly one day your boss now wants to see some dashboards.And he wants to see them in real time! Agregation, Statistics, or any monthly report he may need. You name it!
You would be out of luck, so let's see a few gotchas:

+ S3 lacks of search feature out of the box, so we could add  Elastic Search on top of the the S3 bucket itself.

+ S3 do not have a proper query API. You could list buckets/files (at a cost) and search/filter on the client side. Hower the List operations is not realiable 100%. If you have a high number of elements on your bucket that is being updated, it's likely that your iteration will miss the objects that were added while reading the list.

You could wing those deficiencies providing structure your data, using Parquet format and Athena/Glue (but again it would cost you a lot of $$)

Clearly on this aspect a solution backed by DynamoDB would have been always a better solution.

## Versioning

S3 supports automatic versioning, so it’s trivially easy to track a history of changes or even revert an object to a previous state. Versioning will cost you more bucks.

Dynamo does not provide object versioning out of the box. You can implement it manually, but it’s difficult to block the modification of old versions.

## Cost

It's difficult to find a clear criteria as it depends a lot on the usage patterns. For example using DynamoDB with queries that do not use the right index can increase a lot the monthly bill.

S3 costs are too highg if you’re storing a lot of small records and your writing rate per day is over the million writes per day.

DynamoDB ends up being significantly cheaper for working with small items.

## Summary

Some pointers that could help taking the final decision.

If the objects you need to work with are considerable big, with no reporting needs or queries and you can live with eventual consistency and your usage/access patterns fall in the one-time direct access to the object itself S3 maybe a good candidate.

If you need to store small bits of structured data, with minimal latency, and potentially need reporting/BI analysis to process groups of objects in atomic transactions, choose DynamoDB.
