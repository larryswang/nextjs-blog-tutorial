---
title: "Design a Notification Service"
subtitle: "Design a Notification Service"
date: "2024-09-07"
---

Notification Service is everywhere, it could be an email coming from subscription, a text alert, or a mobile phone app push notification. Today we are going to design a notification service that is one very common question to be asked during an interview session.

## Problem Statement
### Functional Requirements

The notification could be triggered by another service, or job, or user. But we don’t need to worry about that.
The notification will call some 3rd party API, such as email service, IOS push notification etc. We will not discuss that as well.
The notifications are for individual users, using some custom notification.

In this chapter, we will keep adding more constraints and use cases, and come with related numbers. We will also keep evolving our service.

## Step 1. A real time simple notification system
### Requirements
We want to start with something very simple, 1000 DAU, each has 10 notifications per day. The push notifications are text only, each with 1KB. 

### Design
We could quickly draw the diagram:

![images/notification-service-1](/images/notification-service-1.jpg)

Due to the size of the service, we could fit all notifications in the single host. The Notification Service will trigger the notification by using 3rd party push service APIs. It is very simple. Note that before the Notification Service, there should be a proxy/API gateway to perform converting, rate limiting or even authorization, etc. We will not discuss it here in detail.

## Step 2. A real time simple notification system that guarantees exact delivery
### Requirements
On top of 1, we want the notification to be delivered to customers exactly once. We could allow some reasonable delay.

### Design

![images/notification-service-2](/images/notification-service-2.jpg)

First step, we add a persistent storage, notification database. Which stores the mapping between notificationId to the notification message. Such as:

| NotificationId  | Message                    |
| ------- | ------------------------------ |
| Id1 | “Hello”               |

The notificationId is the key to ensure idempotency. When a notification sending event is triggered, it first writes the message to the database, then calls 3rd Party Push Service to send the message. But is this good enough?

Does this guarantee at most 1 delivery? Yes. If another notification is sent to the Notification Service, then the service will find one entry in the database, and it will be simply no-op.
Does this guarantee exactly 1 delivery? No. Let’s say we write the entry in the database, but then the call to 3rd Party Push Service failed. Then the message is never delivered to the customer.

We could resolve this by adding a queue in the Notification Database.

![images/notification-service-3](/images/notification-service-3.jpg)

The Notification Service will write the notification in the database along with the queue message in the same transaction. Then later the queue receiver will pick an item from the queue and call Push Service. This could guarantee exactly one delivery. The queue will have the following schema:

| NotificationId  | Payload                   |
| ------- | ------------------------------ |
| Id1 | empty            |

## Step 3. A real time simple notification system that guarantees exact delivery at large scale
### Requirements
On top of 2, we want to design a service that could scale to serve O(millions) of users.

### Design
The diagram above still applies when it comes to distributed systems. The Notification System could receive the request by Load Balancer (not shown in the diagram) in one of its servers, the server will write the message in Notification Database and Message Queue in the same transaction. The rest should remain the same. 

The Notification Database and Message Queue are both shared by NotificationId, because they have the same primary key in this case. However, if we want to use two separate components in a distributed system, such as AWS RDS + AWS SQS. Then it is possible that the message will not be delivered exactly once. This is unavoidable because of the nature of the distributed system, see [here](https://bravenewgeek.com/you-cannot-have-exactly-once-delivery/). One mitigation is to use some monitoring system. When we push notification to the database/message queue, we should rely on retry and log for debugging.

## Step 4. A real time notification system that pushes notification to all users/a group of users
### Requirements
On top of 3, we want to deliver the same notification to all users/ a group of users.

### Design

![images/notification-service-4](/images/notification-service-4.jpg)

We can tweak the design in step 3 to accommodate this solution. We have a User Service that stores the information of users that we want to send to. Its data could look like:

| UserId  | Metadata                   |
| ------- | ------------------------------ |
| Id1 |  -            |

The workflow could look like:
Some external event triggers the notification to send a message to all users
Notification Service calls User Service to list userIds by page
For each page, write the messages to Notification Database, and write the tasks to Message Queue, before calling the User Service to fetch another page
If the call to User Service failed, then on retry, we will skip enqueuing the messages that are already in Notification Database.

## Conclusion
In this design, we evolved our system to millions of users, with the ability to send notifications to all users step by step. 
