---
layout: post
title: "Reliable Notification System"
author: "Viktor"
tags: [AWS, DynamoDB, Kinesis, Lambda, SES]
---

Almost any application aims to increase engagement by sending notifications to its users.
LinkedIn sends a notification when someone sees your profile,
Facebook, when someone sends you a friend request,
AngelList when your job application has been updated, and so on.

Let's design such a system enabling great marketing capabilities for an application we are working on.

Requirements:
```
Events per second: 10,000 
Notifications type: Email (with easy extensibility to SMS, push notifications, and more)
Events source: DynamoDB
Every event results in an exactly-one notification
```

## Analysing the requirements

### Load

Considering the number of events per second, we will need a pretty scalable system sending over
860,400,000 notifications a day.

We can safely assume that each event and notification message will be relatively small - no more than
a couple of Kbs, Let's say 5Kb. 

This results in 50 Mb/s (400 MB/s) bandwidth, which is no problem for the [AWS and EC2 Network capabilities](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-network-bandwidth.html){:target="_blank"}

### Notifications type

Initially, we want to send email notifications to the users of our application, but we want to support easy
extensibility with SMS and push notifications, for example.

This means we will need a computational unit ( e.g. server ) to decide and broadcast the event notification and be
easy to modify and extend.

## DynamoDB Streams

Events are being produced and stored in DynamoDB, as we already know. So we need a way to consume and process them.

DynamoDB Streams is a convenient way to get notified when a change has happened in a DynamoDB table.
We can subscribe for any of CREATE, UPDATE and DELETE events.

Based on AWS Kinesis DynamoDB Streams provide high performance and reliability. According to their [documentation](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Streams.html){:target="_blank"} 

**DynamoDB Streams helps ensure the following:**

* **Each stream record appears exactly once in the stream.**
    
* **For each item modified in a DynamoDB table, the stream records appear in the same sequence as the actual modifications to the item.**

Which meets the exactly-once delivery criteria in the requirements of our application.

## EC2

We can start thinking of running a server app deployed on an EC2 instance consuming the events and producing notifications, and it would not be a bad solution at all, but 
is a dedicated server actually necessary?

Having a dedicated server comes with all the problems of keeping the server alive, ensuring up-time, updating and deploying the app without any downtime, and so on.

## Lambda

This system is asynchronous by design as we are getting the events, and no one waits to check what happens to them.
We can leverage that by utilizing AWS Lambda as the computational unit we mentioned.

At this point, maybe you wonder why use EC2 or Lambda at all and not just filter-based SNS forwarding messages to the Simple Email Service?

Because having a dedicated computational unit in the pipeline allows greater flexibility and mitigates the unreliable nature of SNS.

Imagine a future case where you want to send a text message to some user via 3rd party SMS service.
An event is being stored in DynamoDB and notified through the table stream to SNS, which then filters it and broadcasts it to the appropriate 3rd party service, which unfortunately turns out to be down. The notification is forever lost!

Having a Lambda allows retrying or waiting for confirmation which ensures the requested from our application reliability and 
allows logging the failed notifications, so they can be later processed and eventually delivered.

![Reliable notifications pipeline](/assets/posts/dynamodb-kinesis-lambda-ses-pipeline.png)

In addition to that Lambda allows easy implementation of programmatic filtering based on complex attributes living in external DBs and services.

And Finally, it takes less to no time to integrate with DynamoDB Streams and plug in the pipeline, so it's worth the effort.

### Why Lambda and not EC2?
* You don't have to manage infrastructure
* It supports easy deployment without downtime, enabling quick updates of business logic and delivery of features.

---

Reviewing the requirements, we can conclude the following:

* Lambda can scale up to 1,000 instances and handle 10,000 events/sec with ease.
* With an appropriate design pattern, we can support many 3rd party services handling the different notification types without compromising the integrity of our service.
* DynamoDB Streams ( Kinesis ) comes with out-of-the-box exactly-once delivery which combined with Lambda satisfies the 'exactly one notification' requirement.

**Thank you for reading!**
