---
title: "Vector Clock"
summary: "A brief summary of Lamport timestamp, Version Vector and Vector Clock"
date: 2022-03-26T17:58:32+08:00
draft: true
---

## causality in distributed system

Now if you have a cluster with multi-clients samultaneously writing to different machine. To avoid conflictions, there must be a total order of events. How to sort the events?

It's an ubiquitous problem in distributed system, let me make a short summary.

## Do not

Never use the physical timestamp as the key to sort. Physical time is not accurate enough among distributed machines, unless the system hang on a while to stride over the accuracy gap, it seems not a good idea.

## 1978 - Lamport timestamp

To sort something, you need a sortable key. So how about a logic timestamp?

*Leslie Lamport* proposed the [*Lamport timestamp*](https://en.wikipedia.org/wiki/Lamport_timestamp) in 1978. It uses a monotonic version number appended with and add it self for each request. For example:

1. A(version=0) -- req --> node(version=1)

2. A(version=1) <-- ack -- node(version=1)

3. B(version=0) -- req --> node(version=2)

4. B(version=2) <-- ack -- node(version=2)

5. C(version=10) -- req --> node(version=11)

6. C(version=11) <-- ack -- node(version=11)

The version of B jump direct from 0 to 2 and the version of the node jump direct from 2 to 11, any event received by the node is ordered. The version is the logic timestamp, any event having a larger timestamp can never impact on events having lower timestamp, thus the causality is gauranteed.

The causality problem is very challenging in distributed systems, the lamport timestamp is just a start. If you expand the logic timestamp to multiple nodes, you will get some consensus algorithms like [*Paxos*](https://en.wikipedia.org/wiki/Paxos_(computer_science)) and [*Raft*](https://en.wikipedia.org/wiki/Raft_(algorithm)).

## 1983 - Version Vector

Now you see the version is a logic timestamp for a serious events. One version number tracks an item, but if there are multiple clients update a value on multiple node samultaneously, how can we merge them(often asynchronously)? (The motivation to do this is performance. Sometimes performance is everything, even at a cost of consistency)

To overwrite the old value by the newer version number? If you do that, the old value will be lost, while the node of which had returned "write ok" to some other client. Thus it comes to *Version Vector*, we use a group of version numbers to track each client's updates, merge them automatically or perform it by the client. Yes, by the client!

- synchronize( (1, 2), (2, 1) ) => conflict, merge to (2, 2) by client defined rule

- synchronize( (1, 2), (2, 2) ) => merge to (2, 2) automatically

further read: CRDT(Conflict-free replicated data type)

## 1988 - Vector Clock

Vector Clock is similar to Version Vector, they share the same core concept -- build a vector to track order of multiple updates. I think the key to distinguish them is to understand the difference between clock and version. Clock is for events, while version is for status.

Before diving in to vector clock, the first question is, why the system have to let multiple clients update a single item simultaneously? How about break the item apart so that a vector clock would be not necessary. Well, there does have some systems did that, like *Cassandra*. You can find some details here: [Why Cassandra Doesn’t Need Vector Clocks | DataStax](https://www.datastax.com/blog/why-cassandra-doesnt-need-vector-clocks).

With a vector clock, the node can give a total order of all the events it received, reaches causality consistency.

![Vector_Clock.svg](https://upload.wikimedia.org/wikipedia/commons/thumb/5/55/Vector_Clock.svg/420px-Vector_Clock.svg.png)

(picture from https://en.wikipedia.org/wiki/Vector_clock)

However, the drawback of a vector clock is it can not resolve conflictions, only figure out them. All the conflictions are delegated to solve by client. A classic example is [*Amazon Dynamo*](https://www.read.seas.harvard.edu/~kohler/class/cs239-w08/decandia07dynamo.pdf) ( not to be confused with [*Amazon DynamoDB*](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Introduction.html) )

## references

1. [Lamport timestamp - Wikipedia](https://en.wikipedia.org/wiki/Lamport_timestamp) 

2. [Version vector - Wikipedia](https://en.wikipedia.org/wiki/Version_vector)

3. [Vector clock - Wikipedia](https://en.wikipedia.org/wiki/Vector_clock)
