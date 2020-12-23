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



