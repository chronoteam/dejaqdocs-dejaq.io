---
title: "DejaQ"
---

Author: Adrian Bledea-Georgescu

Mainteners: Mihai Oprea, Sandu Samyr David

Contributors: Radu Viorel Cosnita, Diana Maftei, Daniel Coman, Silviu Badea

Abstract
--------

DejaQ is a distributed messaging queue built for high-throughput persistent messages that have an arbitrary or time-based order. It allows a point-to-point scheduled message-oriented async communication for loosely coupled systems in time and space (reference). Its main use case is to allow consumers to process messages that are produced out of order (the messages are consumed in a different order than they were produced).

Introduction and topics
-----------------------

DejaQ started as a learning project, but we soon realized that to tackle this problem we need more than one weekend. Taking on the challenge, we pursued and tackled each issue we encountered until we found a simple but scalable solution.

We identified a few kinds of message workflows and each of them will be implemented using a different kind of Topic (collection of messages).

1.  To address the actions based on Timestamps we will build the Timeline topic, that stores the out-of-order produced messages into time-ordered storage, while the consumers “walk” through the timeline as time goes by.
2.  The second type of out-of-order messages are priority-based messages. Similar to the timeline, producers keep adding messages with different priorities. While the Timeline is dependent on Chronology (time) and in most cases has looser consuming requirements, the PriorityQueue will have a more real-time nature and a smaller cardinality of timeline values (eg: 100 distinct priorities compared to billions of seconds of a timeline).
3.  The last type of messages DejaQ will address is the time-recurrent messages. CronMessages topic will be used as a cronjob managed system, but having all the DejaQ characteristics: consistency, accuracy, scalability, and reliability.

Premises
--------

We started the project based on the following assumptions about the users of DejaQ and their needs:

*   Messages have to be processed in a specific order specified by the producers (NOT in the order they were created)
*   The time-sensitive messages should only be processed once (as in by only one service, so instead of supporting a complex publish-subscribe system, we chose a simpler point-to-point architecture).
*   The most important aspect of time-actions is accuracy
*   Consumers are decoupled from producers in Time and Space
*   You have one or more Producers and only 1 group of consumers
*   Messages are consumed async and have a lifespan of at least a few minutes
*   Payloads are immutable (but the metadata can be modified, eg: its position in the queue/timeline)

If your app needs a Pub/Sub messaging queue, CommitLog, Stream processing or an Event Sourcing type of messaging DejaQ is not the best solution for you.

DejaQ can also be used between 2 other messaging queues (eg: 2 kafka topics) to act as a reorder buffer, based on their priorities, or release the messages at a specific time.

Design principles
-----------------

1.  Consistency: although it conflicts with the performance and accuracy, we have to do atomic and persistent write operations (ACID) that leads to stronger -consistency at the storage level and stale data detection at the broker and driver level. Scheduled messages are usually one time transactions and we have to make sure there are no duplicates, misses or lost data. DejaQ is a C (in CAP). This choice will harm the High-Availability and write-performance of the system.
2.  Accuracy: (precision on the timeline topics) we strive to deliver the message to a consumer at the exact Timestamp the producer stated. (hopefully at a second delay, but never before)
3.  Scalability: all layers will scale horizontally: producers, storage, brokers and consumers.
4.  Reliability: the brokers especially have to withstand and be available for as long as they could. Partial degradation will be applied to more levels. This leads to a mandatory high quality of the system, from design to recovery scenarios and failovers.
5.  Delivery guarantees: although not in the first 3 priorities, the system chooses at-least-once-delivery, and strives to deliver only-once. As in all systems, this is a shared responsibility with the user.

More details on the principles DejaQ will be available in the System design documents.

The priority queue
------------

This is the first and simplest topic type that DejaQ will support.

It is a **Durable, Consistent, High Available** queue that supports concurrent producers and consumers. It supports priorities ranging from 0 to 65535.

Alpha version
------------

We have high hopes for this project but in the first iteration, this alpha version we will only deliver an MVP that will contain:

* `Priority Queue` with uint16 priorities (only one type of topic)
* `High availability` You can run the broker in 3 nodes (or more 5,7...to be fault-tolerant to 1, 2 ... nodes)
* `Durability` All nodes will have the entire dataset saved on disk (using Raft for consensus and BadgerDB for embedded storage)
* `Consistency`: The writes will be acknowledged by the majority of the nodes and reads only served by the nodes that are up to date. Using Raft and BadgerDB we can provide a snapshot consistency.
* `One Binary`: All you need to run is a single binary that will contain all the dependencies and logic
* `Easy to use API`: the first API will be HTTP based Swagger/OpenAPI compatible
* _theoretically_ unlimited topics with unlimited partitions, producers, and consumers
* `Dynamic consuming` One of the largest advantages of DejaQ is that topics are split into partitions (internally) and they are assigned dynamically at runtime. Although the algorithm in this alpha version is basic it will be extended in the future to allow more complex consuming strategies.
* `At least once delivery`, the best effort for exactly-once-delivery with the comment that the consumer has to successfully ack the processed messages in a timely manner (as long as it has its partitions assigned)
* `Ordering` is guaranteed at the consumer level (it will receive the smallest priority messages each pull from its assigned partitions). Different consumers may receive higher priority messages at the same time (because they consume from different partitions)
* `shared nothing` architecture - all brokers can serve all purposes (and if not they will act as a proxy)
* `Cloud native` we have docker images and binaries build pipelines, offer support for K8S deployments and export Prometheus compatible metrics. Basically anything you need to run DejaQ in an elastic distributed nature.


The Timeline
------------

This type of topic will **not be delivered** in the first version of DejaQ.
The main use case is the need for generating time-based events like push notifications, marketing campaigns, and re-engagement messages.Most of the time, this scenario was solved with a good-enough solution, most likely a simple cronjob that triggers a time-window algorithm, eg: every 10 minutes process the messages with the timestamp in the span of the last 10 minutes (or from the last run).

We consider that it is better to move the “load” at the produce time, and insert the message on a mutable Timeline. There are more advantages to doing so, but the main one is that Consumers will have a predictable load and latency, allowing the system overall to scale if needed.

The messages are spread over a timeline based on their timestamps. Producers choose the initial position of the message on the timeline (in the future) and will have the absolute right on the messagethey generate for its entire lifespan (move, delete). (the payload is immutable but the metadata is not).

The broker will push all the available messages (that have the timestamp property in the past<= NOW) to a free and healthy consumer. The broker has the shared responsibility with the driver to deliver the messages to a consumer at the exact Timestamp (or after, hopefully with a minimum single ms digit latency), but never before.


Implementation
--------------

We are confident that Go is a good solution to this problem. The broker, tests and the first official driver will be written in Go. Later on, we will have official support for Java and Python clients, with the help of the community.

Observability is another important aspect of DejaQ, the driver will be as transparent as it can be, and the Broker will expose all its inner metrics in a Prometheus\-compatible format.

Conclusion
----------

We are embarking on a long and harsh journey. If you love technical challenges and distributed systems please join us! We are looking for help at all chapters, from owners to testers, from infrastructure to users that need this kind of product.
Contact us: https://github.com/dejaq or team@dejaq.io

