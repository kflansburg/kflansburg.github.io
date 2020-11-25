---
title: "Ordered Stateful Streaming"
date: 2020-11-21T11:39:16-06:00
draft: true
tags: 
 - Spark
 - Streaming
 - Scala
---


I recently spent around two weeks worth of evenings tracking down a subtle bug 
in a Spark Structured Streaming app. In this post I'd like to document what 
I've learned over several years of sporadic work on this project, discuss the 
state of structured streaming frameworks, and share some thoughts I have for 
improvements going forward.

<!--more-->

## Problem Statement

Let's say that you want to stream updates from a constantly changing set
of collection points. Each collection point produces a table of data containing
hundreds of rows or more. To avoid publishing the full table on each update, 
and consuming enormous bandwidth, you follow the common `snapshot/update` 
pattern. This pattern involves first sending a full snapshot of the table,
sometimes called a state-of-the-world message, and then only sending updates
to individual rows thereafter. This pattern is used widely across many 
industries.

The Kafka API integrates nicely with this use case. Each collection site can
produce messages to its own topic. Within a capture session, messages are 
produced sequentially and keyed by session id. This ensures that messages in
a given session are assigned to the same partition, and Kafka guarantees 
ordering within a partition. Finally, topics are named following a common
prefix, `my-topic-prefix-*`, allowing consumers to discover topics and 
subscribe by RegEx pattern.

You would like to perform analytics on this data. This first means 
reconstructing session state. To do this, you must apply an ordered stateful
transformation to each partition independently. While each partition must
be processed sequentially, the entire job can be parallelized up to the total
number of partitions.    

[![Architecture Overview](/streaming-arch.svg)](/streaming-arch.svg)

While this is not *the* most common task in streaming analytics, I believe it
is still a fairly important use case. In particular, this specifically follows
the intended use and behavior of the Kafka API, and so I would expect that
frameworks intended to consume Kafka as a major feature would work within these
assumptions.

## Detailed Requirements

When I set out to build this pipeline several years ago, I spent a considerable
amount of time identifying requirements for this pipeline to operate reliably.
I would like to share them here, as I believe that they are applicable to 
most ordered, stateful streaming applications.

### Strongly Typed Language

I believe this is critical for most long-running pipelines, and makes them much 
easier to support. Using a strongly typed language reduces the likelihood of 
runtime errors which can be devastating to debug months down the road. It also
*increases* development velocity because bugs are caught sooner and you can
deploy to production more confidently.

### Mature Project

Stateful streaming still feels like an emerging technology, there are many 
features in active development across the landscape of streaming frameworks.
I believe that, for now, it makes sense to prefer a mature framework over 
newer ones with exciting features. This is because there is a large initial
hurdle of complexity to support reliable streaming, and many frameworks may
still be tackling this.

### Scalable

For me, scalability in the context of streaming means that a framework can 
distribute tasks across nodes to the same degree that Kafka (or whatever 
message broker you select) can. Data engineers (ideally) take a lot of care to 
select a keying pattern to maximize parallelism within Kafka. When done 
correctly, this can allow Kafka to scale quite well and handle enormous total
throughput. 

### Topic Discovery

Streaming data is frequently distributed across many topics. Given that topics
can already be partitioned, I believe that this should primarily be done for
ergonomics. For instance, if a human wants to view all messages from a given
source, they should be able to consume a single topic, without having to filter
out messages from other sources. To support an ever-changing set of data 
sources, our framework should periodically detect new topics (and prune 
inactive topics). 


### Stream Integrity

There are several tiers of integrity requirements which I believe a framework
should support. Each one has different implications for parallelization. This
should be explicitly specified and guaranteed by the framework. When a loss of
integrity occurs outside of the framework's control (from Kafka or the 
Producer), the job should fail as it will begin producing invalid and undefined
messages itself.

 * **No Integrity:** Useful for fuzzy all-time aggregation metrics. Messages 
   can arrive out of order or not at all.
 * **Fuzzy Window:** Useful for windowed aggregation metrics. Messages can 
   arrive out of order, within some lateness period, and possibly not at all.
 * **Strict Ordering:** Messages must arrive in order, and cannot be skipped.

### Arbitrary Stateful Transformations

Stateful transformations *typically* involve custom business logic. Beyond 
this, I have found built-in primitives for stateful transformations to be not
nearly as complete as, for instance, data frame APIs. Frameworks should
support arbitrary functions applied to each record, as well as arbitrary (but
serializable) state types which are automatically tracked for the user. 

### State Snapshots and Recovery

As workloads become more ephemeral, the ability of streaming applications to
reliably recover from application restarts becomes more critical. Even in 
stateless applications, the framework needs to track messages consumed 
closely in order to prevent reprocessing and retransmission of messages. In
stateful applications, the framework must also periodically create checkpoints 
of the user-defined state. State should be recorded to a reliable storage 
medium (HDFS is common), and care must be taken to ensure the integrity of this 
data and avoid race conditions between checkpoint data and output data from the
job. 

### Sink to Popular Formats and Locations

Many streaming frameworks are only as useful as the systems and formats that
they help to connect. Offering a wide range of output formats and connectivity
is a key focus of many of these frameworks. There is added complexity when 
bridging gaps between streaming and non-streaming formats. For instance, in my
case the goal was to output to a Parquet dataset in S3. Obviously reliable and
fast S3 support was important, as well as a feature-complete Parquet library 
with support for partitioning and compression. 

Complexity lies in determining how frequently to commit batches. Too often, and 
you will get thousands of files per day which is very inefficient for querying. 
Not often enough and you will have massive end-to-end latency for your 
pipeline, as well as substantial reprocessing time in the event of a lost 
batch. Some mitigations exist, such as adding a second file compaction job, or
outputting records in update mode (overwrite entire chunks of the table) rather 
than append mode (only add new records). These have their own complexities, in 
particular they can impose limitations on the business logic of your queries, 
struggle with high variation in throughput between output partitions, or 
greatly increase the compute expense of your pipeline.  

### Latency and Throughput

Many of these frameworks offer latency on the order of milliseconds, but there
are always tradeoffs. As mentioned above, when sinking to non-streaming 
formats, you may want to artificially delay the output of data to produce 
reasonable file sizes. Similarly, some queries rely on windowed processing of
data, the period of which may need to be tuned to match latency needs. 

There are also a number of practical considerations. A major one is the 
location of the producer, broker, and consumer relative to each other
geographically. Your streaming query may be sub-millisecond end-to-end, but
there may be an unavoidable network latency between the job and the broker. 
Even within a single cloud region, you may find between 1 and 
10 milliseconds of latency between nodes in a high-availability configuration. 
When possible, you should use zone or even node affinity controls to place your 
streaming query close to your Kafka brokers.

At the extreme end you may be attempting to aggregate information globally. 
I have found an upper limit (after extensive tuning) of about 2 GB / s 
throughput on a single Kafka partition when dealing with ~160ms network latency 
between two sides of the world. I recommend centralizing the Kafka cluster and 
streaming query, and focusing tuning efforts on the globally distributed 
producers. These situations sometimes call for the use of UDP instead of TCP, 
which can cut latency in half. Unfortunately UDP support amongst message 
brokering applications is poor. It would be great to see these platforms adopt 
QUIC or another multiplexed UDP protocol.

For my use case, latency was not a critical consideration, and I preferred to
wait up to an hour to favor larger Parquet files. 

## Selecting a Framework 

As you can see, there is a lot of complexity that goes into streaming 
applications, and in particular stateful ones. What seems like a 
straightforward use-case results in a laundry list of complex requirements. At 
the time, the use of Kafka and Parquet meant that I would likely need to use an 
Apache project framework. The main frameworks I recall looking into were Kafka 
Streams, Flink, and Spark.

### Kafka Streams

Kafka Streams is appealing because it offers tight integration with the Kafka
API. Specifically, it launches one stateful task per Kafka partition, which 
consumes messages in order from that partition. Parallelism is controlled by
the number of partitions used. Streams also has first-class support for 
aggregating a stream of updates as its table dual. There were a number of
items, however, which concerned me. First, it features a very flexible 
deployment model, but at the time I was not interested in manually configuring 
Yarn or Kubernetes to deploy my application. Second, it is highly Kafka 
specific and, while I generally want to stick with a Kafkaesque API, I was not
certain I wanted to use Kafka itself forever. Finally, Kafka Streams supports
output to Kafka exclusively. This was a deal breaker as the output records of
my application were quite large, making running them through Kafka cumbersome, 
and I would *still* need to run Spark or something to transform these records
to Parquet. 

### Flink

Flink is perhaps the most ambitious of the streaming frameworks I reviewed, 
with many interesting and novel features and the most flexible architecture. 
Flink includes a library for Stateful Functions, which allows you to piece 
together an arbitrary graph of stateful actors. Ordering semantics between
Kafka consumers and these function invocations, however was not made explicit,
and gave me cause for concern. I also felt that it was too immature of a 
project at the time, and I was not convinced by the documentation that it fully 
addressed the other correctness guarantees that I required. I decided not to go 
with Flink, but keep an eye on the project for future work.

### Spark 

In comparison, Spark, and in particular Structured Streaming, had a very 
thorough documentation page which spoke directly to many of my concerns. It 
addressed topic auto-discovery, snapshot and recovery semantics, and of course
Spark supports a broad range of output formats. It was clear that sufficient
thought and engineering had gone into these critical pieces.  

#### Strongly Typed

Spark can be developed in Java or Scala, giving the benefit of strong 
compile-time type checking. Many users use the `DataFrame` API, either 
explicitly or implicitly using Spark SQL queries. A `DataFrame` contains rows 
of type `Row`, which infers schema at runtime. This prevents the compiler from 
verifying that the transformations that you are applying are valid (that the 
columns exist and are of the correct type). 

It is possible, however, to operate on a `Dataset[T]`, where `T` is a 
first-class type. A `DataFrame` is actually equivalent to a `Dataset[Row]`. A 
distinction that is often overlooked (although is hardly a secret) by Spark 
developers is that of type-preserving vs. non-preserving transformations. These 
are listed separately in the [Scala documentation for Datasets](https://spark.apache.org/docs/latest/api/scala/org/apache/spark/sql/Dataset.html). 
As you can see, there are many methods on `Dataset[T]` which inadvertently 
return `Dataset[Row]`, because they are unable to infer the output type. One 
must be careful to avoid these methods, to ensure that your transformation is 
strongly typed end-to-end.   

#### Scalable and Mature

Spark is certainly a mature project, it is under active development, widely 
used, and supported by a number of vendors. Structured Streaming, however is
a somewhat new feature, and is in fact not the first attempt at stream handling in
Spark. Spark is undeniably scalable, and there is a lot of documentation on 
configuration and tuning of Spark clusters for scalability. Indeed, my own 
previous expertise with administering Spark clusters biased my decision here.
There is some  opaqueness, however, surrounding the behavior and parallelism 
that Structured Streaming itself uses, but I was relatively confident that I 
would be able to horizontally scale my job. I was also interested in the 
ability to do large-scale, non-streaming analytics downstream from state 
reconstruction. Spark seemed to be the best option out of the three for 
integrating these workloads into a single cluster or even query. 
 
#### Arbitrary Stateful Transformations

A major feature that Structured Streaming calls out is its support for 
arbitrary stateful transformations. I found this to be fairly flexible and
easy to implement:

```scala
def updateState(
    key: String,
    inputs: Iterator[KafkaMessage],
    oldState: GroupState[MyStateType]
): Iterator[OutputRecord] = {
   // Expire groups (capture sessions) after timeout.
   if (oldState.hasTimedOut) {
     oldState.remove()
     Iterator()
   } else {
       // Initialize state on first call for the session, then retrieve state 
       // from API on subsequent calls. 
       var state = if (oldState.exists) {
           oldState.get
       } else {
           new MyStateType
       }
       // Efficient Iterator over records in batch.
       val result = inputs.flatMap(state.handle_event);
       // Update stored state and reset its timeout.
       oldState.update(state)
       oldState.setTimeoutDuration("1 hour")
       result
   }
}
```
```scala
val query = df
    .groupByKey(msg.session_id)
    .flatMapGroupsWithState(
        OutputMode.Append,
        GroupStateTimeout.ProcessingTimeTimeout
    )(updateState);
```

Notice, however, that there is one catch. The only option for this type of 
transformation is when using Spark's 
[flatMapGroupsWithState](https://spark.apache.org/docs/latest/api/java/org/apache/spark/sql/streaming/GroupState.html) 
method on a dataset that has been grouped by a key. This is frustrating for a 
few reasons. First, what if I don't want to group this operation? There appears 
to be no option to skip this step. Second, `groupByKey` is 
[explicitly called out as something to avoid](https://databricks.gitbooks.io/databricks-spark-knowledge-base/content/best_practices/prefer_reducebykey_over_groupbykey.html)
in nearly every place it is mentioned online. This is because it performs a 
full shuffle of the data. In my case, the grouping was being done by Kafka key, 
so it seemed like it would *already be grouped* as it streams from the 
consumer. 

Structured streaming appears to assume by default that you are trying to 
perform a join operation on topics, and that any grouping being done is 
unrelated to how messages are already grouped by topic and partition. I would 
argue that this is not a reasonable default. Furthermore, the parallelism model 
of your pipeline becomes unclear, because it is no longer necessarily based on 
the parallelism present in the Kafka source. Sadly this appears to be 
unavoidable when subscribing to multiple topics, and in particular using topic 
discovery. This seemed like an inconvenience, but I accepted it given the other 
features that Spark had.  

#### Latency

One of the main limitations of Structured Streaming that is often 
discussed is that it operates on "microbatches". In other words, it is 
not strictly speaking a streaming platform but instead periodically processes
data in batches as (mostly) normal Spark queries. This suited me just fine, 
as I could play with the batch interval to achieve the tradeoff between 
Parquet file size and latency that I was concerned with. If latency is a 
critical focus of your application, I still believe that Structured Streaming
is a solid solution with its 
[Continuous](https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html#continuous-processing) 
trigger mode. I would also argue that periodically (every ~100ms) processing 
new messages in batches is *more* efficient than processing each message 
individually as it arrives, because there is some overhead in managing 
application state each time progress is made.

#### Final Design

In the end, I wrote the following query in Scala. I packaged this query as a 
fat JAR using [sbt assmebly](https://github.com/sbt/sbt-assembly), and 
[deployed it on AWS EMR](https://aws.amazon.com/blogs/big-data/submitting-user-applications-with-spark-submit/). 

```scala
// Contruct a streaming dataset. 
val df = spark.readStream.format("kafka")
  .option("kafka.bootstrap.servers", args(0))
  .option("subscribePattern", "my-topic-prefix-.+")
  // Read from beginning
  .option("startingOffsets", "earliest")
  // Detect missing (expired) offsets
  .option("failOnDataLoss", true)
  .load();

// Parse JSON payload and convert to strongly-typed `DataSet<KafkaMessage>`.
// Here `KafkaMessage` is a case class that I have defined, and I use a UDF
// to parse and validate the payload. Everything beyond this query is strongly
// types. 
val stream = df.select(
      col("topic"),
      col("partition"),
      col("offset"),
      col("key").as[String],
      Parse.parse_json_udf(col("value").as[String]).alias("value") 
  ).as[KafkaMessage]

val query = stream
    // Process each session independently.
   .groupByKey(_.key)
   // Apply stateful transformation using a custom function `updateState`.
   .flatMapGroupsWithState(
       OutputMode.Append,
       GroupStateTimeout.ProcessingTimeTimeout
   )(updateState)
   .writeStream
   .format("parquet")
   .option("path", args(2))
   .option("checkpointLocation", args(3))
   .partitionBy(...)
   .trigger(Trigger.ProcessingTime("15 minutes"))
   .start()

query.awaitTermination();
```

## Practical Difficulties

While I just made the development and deployment process sound straightforward,
there were a number of time-consuming practical issues that I encountered while
developing for streaming and Spark in general. Many of these may be common 
issues which betray a novice understanding of the Java ecosystem, but hopefully
this can help other novices, and from what I've seen, I have little interest in 
spending my time to learn more about Java. 

**Versioning and JAR Packaging**

I personally find dependency management in Spark and Java to be quite 
aggrivating. It is not clear to me how more sophisticated Java package managers
are intended to fit into the Spark development lifecycle. It is quite common to 
need a newer version of Hadoop than your vendored cluster provides. On top of 
this, each vendor has likely deployed a custom fork of certain packages, 
altering their behavior from public documentation. Furthermore, I have found it 
incredibly difficult to even track down which features are supported by which 
version of a package, due to dead legacy documentation links (for actively used 
versions!). 

For this project, I required `hadoop-aws` 2.8, and `spark-sql-kafka-0-10`, 
which enables reading from Kafka and is not bundled by default. It seemed like
the best practice for this (to avoid customizing the cluster you are deploying
into) was to package a "fat" JAR, which includes these depedency in your 
application JAR. In Scala-land, the most common tool for this appears to be the
`assembly` plugin for `sbt`.

I quickly found that packaging files together into a single JAR is problematic
due to conflicting filenames. `assembly` provides a `case` based approach to
resolving these conflicts, but there were so many and it wasn't clear what the
appropriate action was for each one. What I recently realized is that it is 
*extremely critical to mark dependencies which you do not want to include* in 
your `build.sbt` file. Otherwise, you will be including all of Spark and Hadoop 
itself in the JAR, causing many more conflicts. This does not eliminate 
conflicts, but does make them manageable. Note the `provided` to indicate 
not to include a package:

```scala
libraryDependencies += "org.apache.spark" %% "spark-sql" % "3.0.1" % "provided"
```

**JVM Memory Issues**

Another frustrating aspect is how memory is managed in this ecosystem. It 
completely boggles the mind that in `$CURRENT_YEAR`, I can have plenty of free
memory on the system, and my program will fail due to misconfiguration of 
esoteric JVM memory settings. I frankly don't care about the reasons for this,
it should be fixed in such a mature ecosystem.

On top of this, stateful streaming in particular appears to struggle with 
memory issues. Despite having specified every setting I could find in the JVM
and Spark (only God knows if these were respected), making sure to give the 
application exactly as much memory as was present on the node (less the hefty
JVM tax), I still encountered out of memory crashes. My best guess is that 
state in particular is unable to be spilled to disk (although it should not
be that large in memory). Eventually I conceded and doubled the size of the
EC2 instances, but it was annoying to have to pay for so much memory, and 
always felt like a ticking timebomb.  

**Recovery**

With the numerous out-of-memory crashes, I had plenty of opportunities to 
stress test the recovery feature. In general, it seems to work as advertised
but it feels like a weak point given the consequences of failure (multi-hour 
gaps in data).

The first practical consideration is Kafka topic retention. The real tradeoff
here is between disk space cost and the SLA of responding to application 
failures. When the processing pipeline goes down, your now on the clock to 
fix it before you start losing unprocessed records. Given the volume of data, I 
found that setting retention between 1 and 3 days was acceptable. An important
factor here becomes reliably notifying a human when the job fails. 

I also discovered that recovery wasn't 100% reliable. Structured Streaming 
makes it clear that S3 is not a valid location to store checkpoint data, 
because it does not provide the consistency guarantees that a real filesystem 
does. What I found, though, was that it appears to still insist on storing 
some metadata in S3, and uses this in combination with the actual checkpoint
to recover the job, resulting in some lost data. In particular, it appears to
look at "batch" numbers within the metadata, and skip batches which have 
already "occurred", which semantically makes no sense to me because batch 
number is meaningless and will not contain the same offsets from job to job.
Luckily this appears to be a rare, and I've managed to mostly avoid it, but I 
consider it to be a bug. 

**Failure Modes**

I quickly discovered that there are really several kinds of failure modes in
streaming jobs. I will list them in order of increasing severity. 

First, there are environmental failures. I define this as anything that causes
a task failure that is unrelated to the data being processed or the job code
itself. My memory issues are an example of this, where settings can simply be
adjusted and the task retried. The insight here is that these are in general 
the only types of failures when snapshot and recovery saves us from.

Next is data integrity errors. This could be:

* data loss such as expired offsets or loss of Kafka broker storage volume
* malformed checkpoints due to filesystem issues, Spark bugs, or catastrophic 
  task failures that result in a mismatch between output data state and 
  checkpoint state
* an issue with the contents of the Kafka messages themselves, either malformed 
  messages or a bug in the producer resulting in invalid state

In my experience these errors result in gaps in data. Critically, the gap will 
be from the last session start until you restart the job and manually restart
the sessions. *Pipelines with strict SLAs should therefore restart sessions at
a fixed interaval to establish an upward bound on data loss*.  

Finally, there are the bugs which keep me up at night. 

Next

I would describe these categories as structural vs. 
environmental. An environmental failure is one such as my memory issues, where
tasks fail, but they can simply be resumed once memory settings are adjusted. 
Structural failures are where   

**EMR Issues**

I eventually decided that EMR was partly responsible for my memory issues, they
seem to run "hot", preferring to occasionally have to rerun failed tasks in 
exchange for a higher memory limit. The issue with this is that streaming jobs 
do not seem to handle lost tasks as elegantly as traditional Spark queries.

I soon became frustrated with the premium I was paying for larger instances
which did not seem necessary, as well as continued trouble with package 
management between EMR and my development environment.  

**Moving to Kubernetes**

Eventually I elected to move to Kubernetes, deploying Spark in standalone mode
using containers that I created myself. This has a number of advantages in my
mind:

* Eliminates Yarn (Kubernetes is now the orchestrator). 
* No requirements over the number and types of nodes I needed. 
* Seemingly less memory overhead and better control.
* Custom built Spark images with exactly the packages I needed, even running 
  the latest version of Spark and Hadoop. 
* There are simple helm charts for deploying these clusters. 

With all of these hurdles out of the way, my memory issues subsided, I finally 
felt good about my pipeline, and things ran smoothly for several months. 

TODO: Diagram of CICD here

## Tragedy Strikes

### Symptoms

### Debugging Process

**Validating Producer**
**Validating Kafka**
**Validating Streaming Job**
**Spark Logging**

### Identifying the Issue

## Lessons Learned

### Improve your Testing

### Error Checking at Every Step

### Validate your Assumptions

made assumptions based on examples
made assumptions that held for small batches

## My Ideal Streaming Framework

### Infinite Retention

### Versioned Streaming Jobs

### More Sophisticated Parallelism

### Break Point Debugging

### Complex Sinking Behavior

## Conclusion
