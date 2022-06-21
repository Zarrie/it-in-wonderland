---
layout: post
title: "Real-time Geofencing"
author: "Viktor"
tags: [AWS, SQS, Lambda, DynamoDB, ALS, Timestream]
---

Let's design a Real-time GPS Monitoring application for a Taxi company with geofencing and alerting support based on a given set of rules.

![Geo-fence](/assets/posts/geofence.png)

*[Geo-fence](https://en.wikipedia.org/wiki/Geo-fence){:target="_blank"} is a virtual polygon representing real-world area.*

We call geofencing any use of a geo-fence e.g. alerting for entering/leaving a geo-fence.

Requirements:
```
Events per second: 1,000
Features: Geofencing, Rule-based Alerting
Required storage of historical events
```

## Analysing the requirements

### Load

We can assume that the payload will be relatively small - a timestamp, GPS data, vehicle id, and a few metadata properties.
Multiplying that by 1,000 per second results in moderately light application.

This gives us certain flexibility for the environment where the application is living. Even a single server might be able to handle that load with ease.

### Computation

A couple of concerns we should consider at this point include:

* How often do we expect the geofencing rules to change?
* How complicated do we expect those rules to be?
* What capabilities do we have to support those computation resources? e.g. do we have the capacity to manage a server, monitor it, etc.

The last point is actually the most important one in the context, and as it usually turns out, the answer is that we don't have such resources, and we want it 
to be ready as quickly as possible without compromising quality and good practices (hopefully).

Based on those requirements, we can default to AWS Lambda as it is a pretty good solution for such computational problems, and in addition to that 
removes the burden of deploying, managing, and monitoring a server.

We should note that Lambda is suitable only because we assume that the rules are computationally cheap (no more than a couple of seconds response time) and are stateless.
In case we must enforce some sophisticated rules requiring state management Lambda would not be such a good option, and we would prefer a classic server that may be hosted on EKS or maybe even Lightsail. 

Lambda allows great flexibility for integrating with any alerting system, conveniently fitting our requirements.

### Storage

#### Temporary storage

Lambda is not 100% reliable, and we must always design our application to withstand the biggest challenges of distributed systems - network failures, application failures, data center failures, and so on.

To provide that resilience, it's a good idea to make clients publish data to a queue instead of directly hitting Lambda.

This allows us to elegantly decouple our application and make it modular, achieving flexibility for future replacement of any component.

![Vehicles publishing to SQS](/assets/posts/vehicles-publishing-to-sqs.png)

#### Persistent storage

In addition to providing some geofencing functionality, we want to be able to store the consumed data in persistent storage.

We have many options for that.

* PostgreSQL has [good support for spatial data](https://postgis.net/){:target="_blank"}. In terms of scalability, it can easily handle
1 year of data ( 31,536,000 * 1000 items )
* DynamoDB is another great alternative that can easily handle the app's scale, though I'm not sure if it's such a great choice if we require having many spatial queries.
* Another option that I found about a couple of months ago, and I was literally fascinated by is [Tile38](https://tile38.com/){:target="_blank"}.
It provides an out-of-the-box solution for storing and querying sophisticated geospatial data with ultra-fast response.
* Depending on the requirements for that persistent storage, it might actually make sense to keep the data in a time series DB like OpenTSDB, or its AWS alternative
Amazon Timestream. This type of storage would better fit if most queries require retrieving a vehicle location within a given time window.

AWS Timestream is a good option for persistent storage, though Tile38 provides both the storage and the geofencing solution, which is a good fit for our app.

If for some reason, we want to use AWS storage instead of Tile38, we can later attach AWS DymamoDB or Timestream to follow Tile38's write-ahead log and replicate it, or use a simple snapshot replication technique, though this is not required in the case.

We can start with Tile38 hosted on AWS, as it provides the functionality and flexibility needed with a significantly simple interface compared to the other solutions.
In addition to that, it eliminates the need for a separate geofencing service and persistent storage as it provides both.

### Alerting

Having the data delivered by SQS, consumed and evaluated by Lambda, and stored in Tile38, which we further use for the geofencing
we have almost covered the requirements except the alerting.

Having Lambda integrated with Tile38, we get instant updates of the position of every vehicle in the storage plus, we can answer questions like
* has a car entered/left a predefined set of polygons
* are there too many vehicles clustered in a particular area

Based on that, we can now utilize Lambda to detect those cases and handle the alert by forwarding an event to a service responsible for that.

![Real Time GPS tracking pipeline](/assets/posts/vehicles-sqs-lambda-tile38-alerting-pipeline.png)

This concludes the design of the application.

We managed to build a fairly simple yet powerful pipeline that can achieve massive scale with high performance and strong durability guarantees provided by all SQS, Lambda, and Tile38.

**Thank you for reading!**