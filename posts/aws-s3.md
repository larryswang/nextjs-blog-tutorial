---
title: "Design an AWS S3"
subtitle: "System design of an object storage like AWS S3"
date: "2024-09-04"
---
AWS Simple Storage Service, or S3, is one of the most popular cloud object storage services. It is simple to use, extremely durable and highly available. In this post, we will try to design an object storage service like S3.

## Disclaimer
Due to my limited experience and knowledge, I can’t cover all aspects of the AWS S3, everything here is just based on my personal experience. In the real world things may work differently.

This is more like a solution that is based on no special knowledge about S3 that one could come up with in an interview session.

## Problem Statement
### Functional Requirements

We want to design an object storage system. We can think of each object as a file of any kind, such as txt, image, or video.The key functionalities are:
Retrieve an object given an object url
Store the object and return the url
List object given a directory

### Non-Functional Requirements
In the world of object storage, durability is the #1 requirement. AWS S3 [claims](https://docs.aws.amazon.com/AmazonS3/latest/userguide/DataDurability.html) to have 99.999999999% durability, which means they will never lose the file stored inside it. Also it is designed to have 99.99% availability. This is roughly 3k seconds down time per year, calculated by
```
0.01 % * 24 * 365 * 3600 = 3153.6
```
A rough estimation is that S3 has 1m DAU. Each user makes 1000 read requests and 100 write requests on average. Which results in ~10k QPS for writing and ~1k QPS for reading, calculated by
```
1m * 1000 / 24 / 60/ 60 = 11.574 k
1m * 100 / 24 / 60 / 60 = 1.157 k
```
Each write request writes 100M of data. We need to store it for an average of 10 years. This is equivalent to
```
100MB * 1m * 100 * 365 * 10 = 36500 PB
```

## High Level Design
We start with something simple, and gradually add more components as we evolve.

### Step 1
Suppose at first, we don’t have too much data, so we can place everything on a single host, we call it Storage Service. The server is basically a single machine with a hard disk, where the client could send the file and retrieve the file from.
Also, we need a proxy layer in between, which could perform identity checks, rate limiting, rpc converting, etc. We will not go into more details in this part.

![images/aws-s3-1](/images/aws-s3-1.jpg)

### Step 2
A modern disk could hold TBs of data, but if we go all the way to PB, we need to split the data. A natural way is to split the object onto multiple hosts, and each host has a part of the data.

![images/aws-s3-2](/images/aws-s3-2.jpg)

Suppose user1 uploads a file, the file is written in server2. Then all read requests from user1 should be routed to server2. This is something called a [sticky session](https://www.imperva.com/learn/availability/sticky-session-persistence-and-cookies/). This can result in difficulties in load balancing. In system design problems, we should almost ALWAYS avoid using the sticky session.

### Step 3

![images/aws-s3-2](/images/aws-s3-3.jpg)

What we could do better is that we can separate the service into two parts: control plane and data plane. In the data plane, there are two services: Zookeeper Service and Storage Service. The Zookeeper Service (or Configuration Service) is in charge of returning the server that has the file. Moreover, the servers in the control plane/Zookeeper Service should be CPU/memory efficient, but could have minimal amount of storage, while the servers in the Storage Service should be disk heavy.

## Detailed Design
### APIs
As discussed above, we will design three APIs, getObject, uploadObject and listObjects.

```
uploadObject(user_auth, encrypted_object) -> url
```
user_auth: a user authentication token, we will not go into the details here.
encrypted_object: the object in binary format, encrypted.
return the object url after writing to the database

```
getObject(user_auth, url) -> encrypted_object
```
Same as above.

```
listObject(user_auth)
```
The object from the list operation. We could support lists by folder, but here we just assume we will return everything for that user.

### Zookeeper Service
The schema of the Zookeeper Service could look like:

| UserId  | ServerId                    |
| ------- | ------------------------------ |
| User1 | Server1                |
| User2 | Server2                |

For each user, we assign a server for it. But there could be an issue, because the size of the users really differ. For example, imagine a user like Apple, the data could be spread between multiple servers, vs a lot of personal users, they add up and take only one server. Is there anything we could do better?


| UserId| FilePath |ServerId                    |
| ------- | -------------|----------------- |
| User1 | Folder1 | Server1                |
| User1 | Folder2/sub1 | Server2                |
| User1 | Folder2/sub2 | Server1                |

An approach here is that, instead of partitioning by userId, we partition based on file path. Here in the database, the primary key is (userId, filePath). When the user is reading, we start from the root of the folder, and check if there is a match in the primary key. For example, if user input a request like:

```
getObject(Folder2/sub1/myfolder/myfile)
```
Then we first check if there is an entry in Zookeeper service about Folder2. In the table above, we won’t see anything, we then check Folder2/sub1, and find it is in server1. We could limit the depth of the nested folder, so that the check in Zookeeper service could be constant time.

### Metadata Service
We need to store and return user metadata, so that we could list all objects within a folder for a user. For example, in the UI

![images/aws-s3-4](/images/aws-s3-4.png)

Image [source](https://www.codejava.net/aws/create-folder-examples).

When storing the metadata, we have the option of using a separate service, or put it in the same database in the Storage Service. Either one has pros and cons. In the real world, I believe AWS will do separate services considering the scale, because AWS is very large, separating the services might give out a clear responsibility of individual teams and scalability. As of consistency, it is accepted because a delay between files in S3 and metadata is expected. The design now become:

![images/aws-s3-5](/images/aws-s3-5.jpg)

In the data plane, we now have three services: Zookeeper Service, Storage Service and MetadataService. Let’s take a look at the Metadata Database Schema:

| UserId| Folder | Content                    |
| ------- | -------------|----------------- |
| User1 | . |  Folder1               |
| User1 | Folder1 | SubFolder1 |
| User1 | Folder1 | SubFolder2 |

The primary key will be (userId, folder, content). In this case, if a user is fetching all content under a folder, for example, “user1/folder1”, we could perform a list operation on prefix (userId, folder) to return all the content by pagination. This is quite scalable, both for users with nested folders and for users with large numbers of folders.

### Race Condition
Since we have separated Control Plane with Data Plane, a problem here is what if two write requests are fired at the same time? This is a very interesting question, but we will not go into details here because it is worth its own article. In short, AWS S3 uses a strong consistency model, which means for the write requests, whoever comes first will win. For more information, please refer to their [documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html#ConsistencyModel).

### Scalability
If you were asked about this S3 design question during an interview, there is a 100% chance you will talk about the scalability issue with the interviewer. Let’s examine our design so far to see if our service could handle a potential growth of the service.

Control Plane: It uses multiple hosts to receive the client requests, so this part is fully scalable.

Data Plane: Data plane stores user data, we have to do data replicas, for sure. But think one step further, we have to replicate the data across multiple data centers. In this case, if one data center is down, then the rest are unaffected. We could keep evolve our design like:

![images/aws-s3-6](/images/aws-s3-6.jpg)

For servers, they could reside in the same region as the customer. For databases, we need to put them into multiple regions. Within each region, the databases are sharded to achieve scalability; across regions, the databases are replicated to achieve redundancy and split read traffic during peak hours. Note that only servers in region1 could serve the write traffic, all other regions could only service read traffic, and asynchronously update their data. We could do this multiple times to achieve better durability.

How does the Zookeeper Service know which Storage Service to route to? One solution is that the Storage Servers keeps heart beating with Zookeeper Service on its disk storage and its status. So that the Zookeeper Service knows which Server to route to. There are also some topics about this, like [consensus](https://www.youtube.com/watch?v=GeGxgmPTe4c). We will not be talking about this is details either.

### Upload an Object
This is also an interesting topic that is worth mentioning. For example, a user is uploading a file that is huge, say O(GB), what if there is an interruption? We can’t let the user upload from the beginning. We need to perform several steps for this:

User call uploadObject API in the client
Client will break the file into multiple pieces, and send to Storage Service only the metadata. The Storage Service could have a light database, to store the information on if a file has been uploaded or not. We don’t need a lot of storage space, as file chunks already uploaded will be removed from the database.

| HashId | Status          |
| ------- | ------------------- |
| Id1 |      Uploading      |
The client will start to send object chunks by chunk. This could be done by firing a series of upload requests in parallel.
Storage Service will receive the request, write the file, and update the status in the single transaction.
If for some reason, the file uploading is failed, then the status will remain uploading. Later requests will only retry those unfinished tasks.

## Conclusion
Designing an AWS S3 is extremely difficult, considering that it is one of the biggest services in AWS. In the real world, the problem is much more complicated than what we have covered so far. We just covered the most basic information on S3 that might be asked during an interview.
1. Service Architecture
2. Scalability
3. File Uploading

