<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [Kafka](#kafka)
  - [Fundamental Kafka Features](#fundamental-kafka-features)
    - [Sequential I/O:](#sequential-io)
    - [Zero Copy Principle:](#zero-copy-principle)
    - [Batch Data and Compression](#batch-data-and-compression)
    - [Horizontally Scaling:](#horizontally-scaling)
    - [Log Compaction](#log-compaction)
- [State in Streaming](#state-in-streaming)
  - [Stateful vs Stateless](#stateful-vs-stateless)
    - [Join example](#join-example)
  - [Manage States](#manage-states)
    - [Remote States](#remote-states)
    - [Fault-tolerant Local States](#fault-tolerant-local-states)
      - [Local Store Failure Handling](#local-store-failure-handling)
    - [Database as Stream](#database-as-stream)
    - [Local Store Pros and Cons](#local-store-pros-and-cons)
  - [Common use cases with local KV store](#common-use-cases-with-local-kv-store)
    - [Windowed aggregation](#windowed-aggregation)
    - [Table-table join](#table-table-join)
    - [Table-stream join](#table-stream-join)
    - [Stream-stream join](#stream-stream-join)
  - [Industrial Application with KV store](#industrial-application-with-kv-store)
    - [Samza](#samza)
    - [Flink](#flink)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Kafka

## Fundamental Kafka Features

https://medium.com/@sunny_81705/what-makes-apache-kafka-so-fast-71b477dcbf0

Kafka supports a high-throughput, highly distributed, fault-tolerant platform with low-latency delivery of messages.

There are couple of techniques which makes Apache Kafka so fast

* Low Latency Message Delivery.
* Batch Data and Compression.
* Horizontally Scaling.

Kafka leverage following techniques to run fast

### Sequential I/O:

Kafka relies heavily on the filesystem for storing and caching messages. There is a general perception that “disks are slow”, which means high seek time. Imagine if we can avoid seek time, we can achieve low latency as low as RAM here. Kafka does this through Sequential I/O.

Fundamental to Kafka is the concept of the log; an append-only, totally ordered data structure.

Below is a diagram demonstrating log stream (queue), where producers append at the end of the log stream in immutable and monotonic fashion and subscribers/consumers can maintain their own pointers to indicate current message processing.

### Zero Copy Principle:

Traditional way works like

What happens when we fetch data from memory and send it over the network.

1. To fetch data from the memory, it copies data from the Kernel Context into the Application Context.
2. To send those data to the Internet, it copies data from the Application Context into the Kernel Context.

As you can see, it’s redundant to copy data between the Kernel context and the Application context, this leads to consumption of CPU cycles and memory bandwidth, resulting in a drop in performance especially when the data volumes are huge. This is exactly what the zero copy  addresses are.

![zero_copy](https://github.com/zhangruiskyline/system/blob/main/images/zero_copy.png)

### Batch Data and Compression

Efficient compression requires compressing multiple messages together rather than compressing each message individually.

Kafka supports this by allowing recursive message sets. A batch of messages can be clumped together compressed and sent to the server in this form. This batch of messages will be written in compressed form and will remain compressed in the log and will only be decompressed by the consumer.

Assuming the bandwidth is 10MB/s, sending 10MB data in one go is much faster than sending 10000 messages one by one(assuming each message takes 100 bytes).Compression will improve the consumer throughput for some decompression cost.

Kafka supports GZIP and Snappy compression protocols

### Horizontally Scaling:

Lets understand what vertical scaling is first. Let's say for a traditional database server when loads get increased, one way to handle this is to add more resources eg: CPU, RAM, SSD etc. This is called Vertical Scaling. It has couple of disadvantages as below:

Every hardware has a limit, one can not scale upwards indefinitely.What if the machine gets down? It usually requires downtime.Horizontal Scaling is solving the same problem by adding more machines.

Kafka has an ability to have thousands of partitions for a single topic spread among thousands of machines means Kafka can handle huge loads.

### Log Compaction

# State in Streaming

https://www.oreilly.com/content/why-local-state-is-a-fundamental-primitive-in-stream-processing/

## Stateful vs Stateless 

If your SQL query contains only filtering and single-row transformations (a simple ```select``` and ```where``` clause, say), then it is stateless. 

However, if your query involves aggregating many rows (a ```group by```) or joining together data from multiple streams, then it must maintain some state in between rows. If you are grouping data by some field and counting, then the state you maintain would be the counts that have accumulated so far in the window you are processing.

### Join example

* Stream Join:

Join an incoming stream of ad clicks with a stream of ad impressions. These two streams are somewhat time aligned, so you might only need to wait for a few minutes after the impression for the matching click to occur. 

* Stream Enrichment

Take an incoming stream of ad clicks and join on attributes about the user (perhaps for use in downstream aggregations). In this case, you want an index of the user attributes so that when a click arrives you can join on the appropriate user attributes.

## Manage States

### Remote States

A common pattern is for the stream processing job to take input records from its input stream, and for each input record make a remote call to a distributed database.

As you can see, the input stream is partitioned over multiple processors, each of which query a remote database. And, of course, since this is a distributed database, it is itself partitioning over multiple machines. 

![stateful_remote](https://github.com/zhangruiskyline/system/blob/main/images/stateful_remote.jpg)

* One possibility is to *co-partition* the database and the input processing, and then move the data to be directly co-located with the processing. That is what I am calling “local state,” which I will describe next.

### Fault-tolerant Local States

Local state is just data that is kept in memory or on disk on the machines doing the processing. The job can query or modify this data in response to its input.


As the job modifies its local store, it logs out these changes to a Kafka topic. This Kafka topic can be used to replay the changes and restore the local state if the machine fails and the process needs to be restarted on a new host. The log is periodically compacted by removing duplicate updates for the same key to keep the log from growing too large. This facility allows high-throughput updates as the changelog is just another Kafka topic and has the same write-performance Kafka does. Reads are equally high performance, as data is stored locally in memory or on disk.

![stateful_local](https://github.com/zhangruiskyline/system/blob/main/images/stateful_local.jpg)

The more state you have, the longer the restore time will be, but keeping, say, a few GBs per process is very practical and still reasonably quick to restore. Since you can horizontally scale the number of processes across a shared cluster, this means you can horizontally scale the total state the job is maintaining.

#### Local Store Failure Handling

* What if node with local store fails and another node takes over the responsibility ? 

For such cases, the local store also needs to be persisted somewhere. One good way is to record the changes in the local store as stream of changes called changelog (or commitlog) in Kafka world. If you are not exactly sure what a changelog is, I highly recommend to go through this post of Kafka creator, Jay Kreps explaining very well in detail: 

https://engineering.linkedin.com/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying

This changelog is written to a replicated Kafka topic internally which can be used to bring up a new node with same local state as the previous one. 

Also this changelog is not allowed to ever grow using a known Kafka feature called Log Compaction (https://kafka.apache.org/documentation/#compaction) which ensures only the latest value is maintained for each key over a period of time.

In this way both Reads and Writes can be made super fast by using local persistent store along with the fault tolerance being guaranteed by the changelog. The approach above may sound specific to Kafka which has emerged as de-facto distributed messaging system for almost every distributed processing system out there but the intent and concept can be applied to other messaging systems as well.



### Database as Stream

One aspect of the usage patterns that may not be immediately obvious is that you can treat databases as being a source of a stream: the stream of row updates. MySQL, HBase, Oracle, Postgres, and MongoDB all give access to the stream of changes in this way. Kafka has good support for this kind of keyed data and can act as a data subscription layer for data coming out of these databases, making it easy to have a lot of processing jobs subscribing to these changes.

Stream processing jobs can then subscribe to these streams and record the subset of data they need to use in their processing into their local state for fast access.

So, our “data enrichment” use case where we wanted to join information about users to a stream of clicks can be implemented by subscribing to the click stream as well as the user account change stream like this:

When the stream processing job gets a user account database update, it pulls out the attributes it uses and writes them to its local key-value store. When it gets a click event, it looks up the user attributes in its local store and outputs the click event with the additional user attributes included.

In fact, it often happens that all the processing is against database change streams coming from different databases. The stream processor acts as a kind of fancy cross-database trigger mechanism to process changes, usually to create some materialized view to be used for serving live queries:


![database_stream](https://github.com/zhangruiskyline/system/blob/main/images/database_stream.jpg)

### Local Store Pros and Cons

- Pros:

* Local state can be indexed and accessed in a variety of rich ways that are much harder with remote databases

inverted index? A compressed bitmap index? Key-value access? Want to do full table scans? Want to compute complex aggregations? You can do all of these

* Local, in-process data access can be much, much faster than remote access

* Local data access is easier to isolate

- Cons

* Duplicate in data

* Restore time for a failed job will become slow if local state too large

* if significant logic around data access is encapsulated in a service, then replicating the data rather than accessing the service will not enforce that logic.

## Common use cases with local KV store

### Windowed aggregation

Example: Counting the number of page views for each user per hour

Implementation: You need two processing stages.

The first one re-partitions the input data by user ID, so that all the events for a particular user are routed to the same stream task. If the input stream is already partitioned by user ID, you can skip this.
The second stage does the counting, using a key-value store that maps a user ID to the running count. For each new event, the job reads the current count for the appropriate user from the store, increments it, and writes it back. When the window is complete (e.g. at the end of an hour), the job iterates over the contents of the store and emits the aggregates to an output stream.

Note that this job effectively pauses at the hour mark to output its results. This is totally fine for system like Samza, as scanning over the contents of the key-value store is quite fast. The input stream is buffered while the job is doing this hourly work.

### Table-table join


Example: Join a table of user profiles to a table of user settings by user_id and emit the joined stream

Implementation: The job subscribes to the change streams for the user profiles database and the user settings database, both partitioned by user_id. The job keeps a key-value store keyed by user_id, which contains the latest profile record and the latest settings record for each user_id. When a new event comes in from either stream, the job looks up the current value in its store, updates the appropriate fields (depending on whether it was a profile update or a settings update), and writes back the new joined record to the store. The changelog of the store doubles as the output stream of the task.

### Table-stream join

Example: Augment a stream of page view events with the user’s ZIP code (perhaps to allow aggregation by zip code in a later stage)

Implementation: The job subscribes to the stream of user profile updates and the stream of page view events. Both streams must be partitioned by user_id. The job maintains a key-value store where the key is the user_id and the value is the user’s ZIP code. Every time the job receives a profile update, it extracts the user’s new ZIP code from the profile update and writes it to the store. Every time it receives a page view event, it reads the zip code for that user from the store, and emits the page view event with an added ZIP code field.

If the next stage needs to aggregate by ZIP code, the ZIP code can be used as the partitioning key of the job’s output stream. That ensures that all the events for the same ZIP code are sent to the same stream partition.

### Stream-stream join

Example: Join a stream of ad clicks to a stream of ad impressions (to link the information on when the ad was shown to the information on when it was clicked)

In this example we assume that each impression of an ad has a unique identifier, e.g. a UUID, and that the same identifier is included in both the impression and the click events. This identifier is used as the join key.

Implementation: Partition the ad click and ad impression streams by the impression ID or user ID (assuming that two events with the same impression ID always have the same user ID). The task keeps two stores, one containing click events and one containing impression events, using the impression ID as key for both stores. When the job receives a click event, it looks for the corresponding impression in the impression store, and vice versa. If a match is found, the joined pair is emitted and the entry is deleted. If no match is found, the event is written to the appropriate store. Periodically the job scans over both stores and deletes any old events that were not matched within the time window of the join.

## Industrial Application with KV store

### Samza

http://samza.incubator.apache.org/learn/documentation/0.7.0/container/state-management.html

### Flink



