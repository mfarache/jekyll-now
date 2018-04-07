---
layout: post
title: Considerations when working with Amazon S3 .
tags: [ S3, AWS, Storage ]
---
We use S3 to store stuff in the cloud, but there are a few things that not everyone maybe aware of. 
The following post is just a scratchpad / notes on some items I have learnt while working with AWS S3 in the last month.

# Simple storage service - The basics

Buckets are similar to folder on your laptop, they keep stuff.

Buckets cannot be nested, instead you use keys. If the key contains a slash separator then it will be behave 
as folders, so you can drill drown across the bucket up and down.

You can use console UI that provides CRUD operations on your buckets, ACL permission sets, and much more things.
Of course everything is also accesible using aws cli.

The following items is a summary of things I learnt recently while using S3.

# Think on your keys

It's always a good idea to distribute your objects within bucket so they are organized in some even way
For example you could create a MD5 hash, and get the first 8 bytes to be the key of your bucket.
Other strategies can be alphabetical or time based (organizing by year/month/etc).
There is no impact on performance though, so if you have 1M objects in a bucket it will be the same as if you had only one.

# Be aware of limitations

Bucket names must be unique across all regions. 
They should be DNS compliant (should not contain uppercase letters or underscores)
S3 bucket names cannot be renamed.
S3 buckets cannot be deleted unless they are empty using API calls.
Up to 100 buckets per account

# Protect your data 

You can set default encryption on a bucket so that all objects are encrypted when they are stored in the bucket.
Encryptionh happens before asset is written. Decryption occurs on download.
There are two types provided out of the box (Amazon S3-managed keys (SSE-S3) , AWS KMS-managed keys (SSE-KMS))

Once we have defined our bucket we can assign them policies about which actions can be performed on the bucket
The typical uses cases allow access to a specific IAM users and define which operations the user can perform on the bucket 

```bash
{
   "Version":"2012-10-17",
   "Statement":[
      {
         "Effect":"Allow",
         "Action":[
            "s3:PutObject",
            "s3:GetObject",
            "s3:GetObjectVersion",
            "s3:DeleteObject",
            "s3:DeleteObjectVersion"
         ],
         "Resource":"arn:aws:s3:::mybucket/Mau/*"
      }
   ]
}
```

Do not ever make your bucket public (unless you have a very good reason to do that)

# Versions

Versioning automatically keeps up with different versions of the same object. 
By default versioning is not enabled. Once you enable it, every time you store the same object in a bucket the old version is still stored in your bucket, and it has a unique Version ID so that you can still see it, download it, or use it in your applications.
Versioning is configured at bucket level.
Deleting objects with versioning enabled, leads to not being able to access your object directly. Instead you can access using version identifier.
There is no way back once you have configured versioning in a bucket.

# Object Lifecycle

S3 provides definition of lifecycle rules in order to determine how your data expires, moves across different storage pricing schemas.
For example if we have versioning enabled we could end up with a bunch of data.
It may result obvious (or not) but versioning imply more costs, as the amount of data grows to keep those versions.
What if is unlikely that you need access to those versions in the short/medium term future? 
We could write a rule that moves our object into Glacier after 30 days and after one year it removes permanently. 

# Amazon S3 Transfer Acceleration

It can be enabled/disabled at bucket level. 
It provides faster transfer of files over long distances (not close to the bucket's region )
It uses Amazon CloudFront edge locations. 
As the data arrives at an edge location, data is routed to Amazon S3 via an optimized network path.

```bash
bucketname.s3-accelerate.amazonaws.com
```

# Requester Pays

It can be enabled/disabled at bucket level.
Usually owner account is who pays the cost of storing and transfering data.
With Request Pays the requester instead of the bucket owner pays the cost of the request and the data download 
Useful to share big data sets, owner should not be paying by downloaders.
Anonymous access to bucket is not permitted.

# Signed URLS

It's a good practice to use AWS SDK libraries that sign S3 URLs based on Amazon S3 supports Signature Version 4.
We need to provide AWS credentials, algorithm and region.
Why we want to do that? 
It's a way to verify the authenticity of the requester - not everyone should have credentials on a AWS user account
It's a way to avoid someone messing with the signed portion of the request - they expire in 15minutes  
It's a way to avoid request tampering, as AWS decodes the request on the server side so they should match.

# Testing S3 in isolation

I was involved recently in a development involving s3 stack. At the very begining I was wondering how to decouple my code of the storage service without having a real S3 store nor AWS account. 

I stumbled with s3Proxy for Java,  that help working with your local filesystem but still using AWS S3 API. There are still many features that are not supported like ACL policies, object tagging. I just needed simple copy across the bucket which worked perfectly.

# Browsing S3 folders like a champion

Let's face it. The UI S3 console sucks. Once you start having lots of files and a nested folder structure it becames a cumbersome task to navigate and reach your file. 

All I want is just drop a file here, or copy from there.

AWS Cli provides the usual suspect calls to copy or move file. But still we need something more agile.
So I've found several alternatives like s3fs, riotfs and goofys. 
The last one is based in Go and seems to be the new Kid of the block.
As I'm personally interested in Go lately I opted for this one

Once you clone the googys repository and follow setup instructions you just need to mount your bucket into your filesystem. 

```bash
$ cat ~/.aws/credentials
[default]
aws_access_key_id = <MY-KEY-GOES-HERE>
aws_secret_access_key = <MY-SECRET-KEY>

$ $GOPATH/bin/goofys <bucket> <mountpoint>
# if you only want to mount objects under a prefix
$ $GOPATH/bin/goofys <bucket:prefix> <mountpoint> 
```

# Useful links

+ [s3Proxy][1]
+ [Goofys][2]
+ [URL shortener github repository][3]
+ [Remote Debugging openFaas apps][4]
+ [Improve logging for functions during erorr][5]
+ [Functions as a service with OpenFaaS][6]

[1]: https://github.com/gaul/s3proxy

