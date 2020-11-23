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

![Architecture Overview](/streaming-arch.svg)

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


**State Snapshots and Recovery:** The pipeline should be able to periodically
snapshot state and seamlessly recover from failure. 

### Sink to Popular Formats and Locations

**Sink to S3:** I wanted to be able to write out batches of data to a 
partitioned Parquet dataset in S3. 

### Latency
**Latency:** I did not have strong latency requirements, up to an hour. I 
favored larger Parquet files over shorter latency.

## Selecting a Framework 

As you can see, there is a lot of complexity that goes into streaming 
applications, and in particular stateful ones. What seems like a 
straightforward use-case results in a laundry list of complex requirements.

The main frameworks I recall looking into were Kafka Streams, Flink, and Spark.
At the time, the use of Kafka and Parquet meant that I would likely need to use
an Apache project framework. Kafka Streams did not appear to be as fully 
featured as the others, and felt too Kafka specific. Flink appeared to be a 
more ambitious project, but was still up and coming. I was enticed by the wide
range of advanced features it described, but was not convinced that it directly
addressed many of my concerns above. In comparison, Spark, and in particular
Structured Streaming, had a very thorough documentation page which directly
discussed many of my concerns. I decided to go with Spark, and keep an eye on
Flink. 

**Strongly Typed** 

Spark can be developed in Java or Scala, giving the benefit of strong 
compile-time type checking. A distinction that is often overlooked by Spark 
developers is that of type-preserving vs. non-preserving transformations. Many 
users use the `DataFrame` API, either explicitly or implicitly using Spark SQL 
queries. A `DataFrame` contains rows of type `Row`, which infers schema at
runtime. This prevents the compiler from verifying that columns you are 
operating are the type that you are expecting, or even exist. 

TODO Dataset capitalization, link to typed transformation docs

It is possible, however, to operate on a `Dataset<T>`, where `T` is a 
first-class type. A `DataFrame` is actually equivalent to a `Dataset<Row>`. The
catch is that there are many methods on `DataSet<T>` which inadvertently return
`Dataset<Row>`, because they are unable to infer the output type. One must be 
careful to avoid these methods, to ensure that your transformation is strongly
typed end-to-end.   

**Scalable and Mature** 

Spark is undeniably scalable. There is some opaqueness surrounding the 
parallelism that Structured Streaming itself uses, but I was confident that
I would be able to horizontally scale my job. Spark is also a fairly mature
project, and was appealing because I already had some experience with 
developing and hosting Spark applications.

**Topic Discovery, Integrity, Snapshots, and Parquet**

As mentioned, Spark Structured Streaming explicitly covered these concerns
in its documentation, giving me a lot of confidence that they had 
well-engineered solutions. My experience to date confirms these assumptions.

**Arbitrary Stateful Stransformations**

TODO Code snippet
TODO Lack of ordering guarantee in readme
TODO Does flink offer a solution? 

A major feature that Structured Streaming calls out is its support for 
arbitrary stateful transformations. I found this to be fairly flexible and
easy to implement, with one catch. The only option for this type of 
transformation is when using Spark's `flatMapGroupsWithState` method on a
dataset that has been grouped by a key. 

This is frustrating for a few reasons. First, what if I dont want to group this
operation? There appears to be no option to skip this step. Second, 
`groupByKey` is explicitly called out as something to avoid in nearly every 
place it is mentioned online. This is because it performs a full shuffle of 
the data. In my case, the grouping was being done by Kafka key (or partition, 
it doesn't matter), so it seemed like it would *already be grouped*. This 
seemed like a mild inconvenience, but I accepted it given the other features 
that Spark had. Finally, the implication of this is that the streaming 
`DataSet` that I was operating on mingles messages from different topics
within a batch. This seems like an undesirable default and makes it somewhat
unclear what kind of parallelism is happening under the hood. Sadly this 
appears to be unavoidable when subscribing to multiple topics, and in 
particular using topic discovery.
 
**Latency**

One of the main limitations of Structured Streaming that is often 
discussed is that it operates on "microbatches". In other words, it is 
not strictly speaking a streaming platform but instead periodically processes
data in batches as (mostly) normal Spark queries. This suited me just fine, 
as I could play with the batch interval to achieve the tradeoff between 
Parquet file size and latency that I was concerned with. 

**Final Design**

In the end, I wrote the following query in Scala. I packaged this query as a 
fat JAR using `sbt assmebly`, and deployed it on AWS EMR. 

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
// to parse and validate the payload.  
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
   // TODO LOOK AT TIMEOUT
   // Apply stateful transformation using a custom function `updateState`.
   .flatMapGroupsWithState(
       OutputMode.Append,
       GroupStateTimeout.NoTimeout
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

**Tragedy Strikes**

NOTES
