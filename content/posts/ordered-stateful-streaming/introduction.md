---
title: "Introduction"
date: 2020-12-18T00:00:00-00:00
description: In the first post of this series on ordered, stateful streaming, I
    will outline the need for this type of transformation, and detailed
    requirements for such an application.
draft: true
toc: false
series:

- "Ordered Stateful Streaming"

tags:

- Spark
- Streaming
- Scala

---

I recently spent two weeks tracking down a subtle bug in a Spark Structured
Streaming application which I have been maintaining for several years. Having
dealt with many such time-consuming bugs over the years, I've decided to
compile my experiences working with ordered, stateful streaming applications
into a series of posts. This series will serve as an introductory guide to the
design and operation of stateful streaming pipelines, and hopefully spur some
further development to simplify this process in the future.

<!--more-->

## Overview

There are [several different classes](#stream-integrity) of stream processing
applications, each with different constraints on how messages are processed.
This can have a significant impact on parallelism and application complexity.
In this series, I will primarily focus on what I consider to be the most
rigorous class of stream processing application: an arbitrary, stateful
transformation is applied to messages exactly in order and without skipping
messages. In this post I will describe the types of processing that lead to
these constraints, and outline detailed requirements for such an application.

## Use Case

Let's say that you want to stream updates from a constantly changing set
of sensors. Each sensor produces a table of data containing hundreds of rows or
more. To avoid publishing the full table on each update, and consuming enormous
bandwidth, you follow the common `snapshot/update` pattern. This pattern
involves first sending a full snapshot of the table, sometimes called a
state-of-the-world message, and then only sending updates to individual rows
thereafter. This pattern is used widely across many industries.

The Kafka API integrates nicely with this use case. Each sensor can produce
messages to its own topic. Within a capture session, messages are produced
sequentially and keyed by session ID. This ensures that messages in a given
session are assigned to the same partition, and Kafka guarantees ordering
within a partition, allowing session state to be reconstructed from the stream
of messages. Finally, topics are named following a common prefix,
`my-topic-prefix-*`, allowing consumers to discover topics and subscribe by
RegEx pattern.

To perform streaming analytics on this data, you must reconstruct the session
state within your stream processing application. To do this, you must apply an
ordered stateful transformation to each partition, independently. While each
partition must be processed sequentially, the entire job can be parallelized up
to the total number of partitions. Below is an illustration of this
architecture, showing how messages are produced to Kafka, and topics are
processed in parallel.

[![Architecture Overview](/streaming-arch.svg)](/streaming-arch.svg)

While this is not *the* most common task in streaming analytics, I believe it
is still a fairly important use case. In particular, this specifically follows
the intended use and behavior of the Kafka API, and so I would expect that
frameworks intended to consume Kafka as a major feature would work within these
assumptions.

## Detailed Requirements

When I set out to build this pipeline several years ago, I spent a considerable
amount of time identifying the requirements for it to operate reliably. I will
share them here, as I believe that they are applicable to most ordered,
stateful streaming applications.

### Strongly Typed Language

I believe being able to use a strongly typed language is critical for
developing long-running pipelines, and makes them much easier to support. Using
a strongly typed language reduces the likelihood of runtime errors which can be
devastating to debug months down the road. It also *increases* development
velocity because bugs are caught sooner and you can deploy to production more
confidently.

### Mature Project

Stateful streaming still feels like an emerging technology, and there are many
features in active development across the landscape of streaming frameworks.
I believe that, for now, it makes sense to prefer a mature framework over
newer ones with exciting features. This is because there is a large initial
hurdle of complexity to support reliable streaming, and younger frameworks may
still be perfecting this.

### Scalability

Scalability in the context of streaming means that a framework can distribute
message processing tasks across nodes to the same degree that Kafka (or
whatever message broker you select) can. Data engineers ideally take a lot of
care to select a keying pattern to maximize parallelism within their message
brokering system. When done correctly, this can allow these systems to scale
quite well and handle enormous total throughput. A streaming application should
be able to match this parallelism and dynamically schedule tasks across
multiple nodes as partitions are created and expire.

### Topic Discovery

Streaming data is frequently distributed across many topics. Given that topics
can already be partitioned, I believe that partitioning messages with the same
schema across multiple topics should primarily be done for ergonomics. For
instance, if a human wants to view all messages from a given source, they
should be able to consume a single topic without having to filter out messages
from other sources. To support an ever-changing set of data sources, frameworks
should periodically detect new topics (and prune inactive topics).

### Stream Integrity

As mentioned above, there are several classes of stream processing
applications, each with different integrity requirements, which I believe any
framework should support. These requirements have different implications for
parallelization. When a loss of integrity occurs outside of the framework's
control (from Kafka or the Producer), the job should fail as it will begin
producing invalid and undefined messages itself. Here is a rough outline of
these classes of integrity requirements:

- **No Integrity:** Messages can arrive out of order and occasionally multiple
  times or not at all. Useful for fuzzy all-time aggregation metrics where some
  inaccuracy can be tolerated.
- **Guaranteed Once Delivery:** Messages can arrive out of order but cannot be
  skipped or repeated. An example would be metered usage billing where the
  order is not important but precise aggregation is.
- **Guaranteed Order:** Messages must arrive in order, but can be skipped or
  repeated. This applies where you simply care about the latest value, and can
  tolerate missed updates. This is trivially implemented using in-message
  sequence numbers.
- **Fuzzy Window:** Messages can arrive out of order, within some lateness
  period. This is useful for windowed aggregation metrics, and many frameworks
  focus on this class.
- **Strict Ordering:** Messages must arrive in order, and cannot be skipped.

### Arbitrary Stateful Transformations

Many stateful transformations involve custom business logic. Beyond this, I
have found the built-in primitives for stateful transformations to be not
nearly as complete as, for instance, data frame APIs. Frameworks should support
arbitrary functions applied to each record, as well as arbitrary (but
serializable) state types which are automatically managed for the user.

### State Snapshots and Recovery

As workloads become more ephemeral, the ability of streaming applications to
reliably recover from application restarts becomes more critical. Even in
stateless applications the framework needs to track messages consumed closely
in order to prevent reprocessing and retransmission of messages. In stateful
applications, the framework must also periodically create checkpoints of the
user-defined state. State checkpoints should be recorded to
[strongly consistent](https://cloud.google.com/storage/docs/consistency)
distributed storage (HDFS is common, S3
[only recently](https://aws.amazon.com/blogs/aws/amazon-s3-update-strong-read-after-write-consistency/)
achieved this). Care must also be taken to ensure the integrity of this data
and avoid race conditions between checkpoint data and output data from the
job.

### Sink to Popular Formats and Locations

Streaming frameworks are only as useful as the systems and formats that they
help to connect. Offering a wide range of output formats and connectivity is a
key focus for many framework authors. There is added complexity when bridging
gaps between streaming and non-streaming formats because it is desirable to
output data in fixed-size chunks with some upper bound on latency, whilst input
data varies in throughput.

For instance, in my case, the goal was to output to a Parquet dataset in S3.
Obviously reliable and fast S3 support was important, as well as a
feature-complete Parquet library with support for partitioning and compression.
The complexity lies in determining how frequently to commit batches. If
committed too often, you will get thousands of files per day which is very
inefficient for querying. Conversely, if committed too infrequently you will
have significant end-to-end latency for your pipeline, as well as substantial
reprocessing time in the event of a lost batch.

Some mitigations exist, such as adding a second file compaction job, or
outputting records in update mode (overwrite entire chunks of the table) rather
than append mode (only add new records). These have their own complexities, in
particular they can impose limitations on the business logic of your queries,
struggle with differences in throughput between output partitions, or greatly
increase the compute expense of your pipeline. Needless to say, there is no
one-size-fits-all tool and I have found these considerations to have a large
impact on the choice of framework.

### Latency and Throughput

Many of these frameworks offer latency on the order of milliseconds, but there
are always tradeoffs. As mentioned above, when writing output to non-streaming
formats you may want to artificially delay the output of data to produce
reasonable file sizes. Similarly, some queries rely on windowed processing of
data, the period of which may need to be tuned to match latency needs.

There are also a number of practical considerations when it comes to total
throughput. A major consideration is the locations of the producer, broker, and
consumer relative to each other geographically. Your streaming query may be
sub-millisecond end-to-end, but there may be an unavoidable network latency
between the job and the producer via the broker. Even within a single cloud
region, you may find between 1 and 10 milliseconds of latency between nodes in
a high-availability configuration. When possible, you should use zone or even
node affinity controls to place your streaming queries close to your message
brokers.

At the extreme end you may be attempting to aggregate information globally.
I have found an upper limit (after extensive tuning) of about 2 GB / s
throughput on a single Kafka partition when dealing with ~160ms network latency
between two sides of the globe. I recommend centralizing the Kafka cluster and
streaming query, and focusing tuning efforts on the globally distributed
producers. These situations sometimes call for the use of UDP instead of TCP,
which can cut latency in half. Unfortunately UDP support among message
brokering applications is poor. It would be great to see these platforms adopt
QUIC or another multiplexed UDP protocol in the future.

For my use case, latency was not a critical consideration, and I preferred to
wait up to an hour to favor larger Parquet files.

## Conclusion

In this post, I outlined key considerations when selecting streaming frameworks
and designing streaming pipelines. In the next post I will discuss several
frameworks which I considered when first designing my pipeline. Each framework
has strengths and weaknesses, and has since improved, but it is useful to
review the process and learn from design decisions that did or did not pay off.
