---
title: "Selecting a Framework"
date: 2020-12-05T00:00:01-00:00
draft: true
toc: false
series:

- "Ordered Stateful Streaming"

tags:

- Spark
- Streaming
- Scala

---

In the second post of this series I explore the strengths and weaknesses of
several popular streaming frameworks. This analysis was performed a couple of
years ago with a particular application in mind. These frameworks have since
improved, but this post should provide some insight into the tradeoffs and
decisions involved when designing streaming applications, and lessons can be
learned from choices that did and did not pay off.

<!--more-->

## Selecting a Framework

As was made clear in the previous post, there is a lot of complexity that goes
into streaming applications, particularly stateful ones. What seems like a
straightforward use-case results in a laundry list of complex requirements. At
the time, the use of Kafka and Parquet meant that I would likely need to use an
Apache project framework. The main frameworks I recall looking into were Kafka
Streams, Samza, Flink, and Spark (which I eventually selected).

### Kafka Streams

Kafka Streams is appealing because it offers tight integration with the Kafka
API. Specifically, it launches one stateful task per Kafka partition, which
consumes messages in order from that partition. Parallelism is controlled by
the number of partitions used. This exactly matches the behavior I desired in
the previous post. Streams also has first-class support for aggregating a
stream of updates as its
[table dual](https://dl.acm.org/doi/10.1145/3242153.3242155).

There were, however, a number of items which concerned me. First, Kafka Streams
features a very flexible deployment model, but at the time I was not interested
in manually configuring Yarn or Kubernetes to deploy my application. Second, it
is highly Kafka-specific and, while I generally want to stick with a Kafkaesque
API, I was not certain I wanted to use Kafka itself forever. Finally, Kafka
Streams supports output to Kafka exclusively. This felt like a deal breaker
because the output records of my application are quite large, making writing
them out to Kafka cumbersome, and I would *still* need to run Spark or
something similar to transform these records to Parquet.

### Apache Samza

Apache Samza was a recent addition to Apache's catalog of stream processing
frameworks. I liked that the documentation was
[very clear](https://samza.apache.org/learn/documentation/0.7.0/introduction/concepts.html)
about the parallelism model used to consume Kafka messages. Like Streams,
however, Samza does not appear to support Parquet output. Samza does support
stateful operations, but state appears to be managed via an API, which would
be very inconvenient for my application. Given that the project had not yet
reached version `1.0`, I felt that it would not be a good choice for my
pipeline.

### Apache Flink

Apache Flink is perhaps the most ambitious of the streaming frameworks that I
reviewed, with many novel features and the most flexible architecture. Flink
offers
[stateful transformations](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/stream/state/state.html)
which appear to support my use case via "Raw" Operator State. Examples of this
use case were unfortunately hard to find, although this has
[since been improved](https://ci.apache.org/projects/flink/flink-docs-release-1.11/dev/libs/state_processor_api.html#operator-state-1).
I felt that, despite a plethora of advanced features, it was difficult to
validate some of my basic requirements using Flink's documentation, and I
decided that the framework was too immature. I chose not to go with Flink, but
to keep an eye on the project for future work.

### Apache Spark

In comparison, Spark, and in particular
[Structured Streaming](https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html)
which was new in Spark 2 and had recently introduced stateful transformations
in Spark 2.3, had a very thorough documentation page which spoke directly to
many of my concerns. It addressed topic auto-discovery, snapshot and recovery
semantics, and of course Spark supports a broad range of output formats. It was
clear that significant thought and engineering had gone into the critical
pieces discussed in the remainder of this post.

#### Strongly Typed

Spark can be developed in Java or Scala, giving the benefit of strong
compile-time type checking. Many users use the `DataFrame` API, either
explicitly or implicitly using Spark SQL queries. A `DataFrame` contains rows
of type `Row`, which infers schema at runtime. This prevents the compiler from
verifying that the transformations that you are applying are valid (i.e. that
the columns exist and are of the correct type).

It is possible, however, to operate on a `Dataset[T]`, where `T` is a
first-class type. A `DataFrame` is actually equivalent to a `Dataset[Row]`. A
distinction that is often overlooked (although is hardly a secret) by Spark
developers is that of type-preserving vs. non-preserving transformations. These
are listed separately in the
[Scala documentation for Datasets](https://spark.apache.org/docs/latest/api/scala/org/apache/spark/sql/Dataset.html).
In this documentation there are many methods on `Dataset[T]` which
inadvertently return `Dataset[Row]` because they are unable to infer the output
type. One must be careful to avoid these methods to ensure that your
transformation is strongly typed end-to-end.

#### Scalable and Mature

Spark is certainly a mature project: it is under active development, widely
used, and supported by a number of vendors. Spark is undeniably scalable, and
there is a lot of documentation on configuration and tuning of Spark clusters
for scalability. Indeed, my own previous expertise with administering Spark
clusters biased my decision here.

Structured Streaming is a somewhat new feature and is in fact not the first
attempt at stream handling in Spark. The older
[DStreams API](https://spark.apache.org/docs/latest/streaming-programming-guide.html)
does not support the `Dataset` API, appears to have weaker end-to-end integrity
guarantees, and no longer appears to be the main focus of development. It does,
however, have its own stateful transformation support, introduced in Spark 1.6,
and appears to support much more granular control of parallelism when consuming
Kafka partitions. I decided to focus my investigation on the newer Structured
Streaming API.

There is some  opaqueness surrounding the behavior and parallelism
that Structured Streaming itself uses, but I was relatively confident that I
would be able to horizontally scale my job. I was also interested in the
ability to do large-scale, non-streaming analytics downstream from state
reconstruction. Spark seemed to be the best option out of the three for
integrating these workloads into a single cluster or even a single query.

#### Stateful Transformations

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
    .groupByKey(msg.key)
    .flatMapGroupsWithState(
        OutputMode.Append,
        GroupStateTimeout.ProcessingTimeTimeout
    )(updateState);
```

Notice, however, that there is one catch. The only option for this type of
transformation is when using Spark's
[flatMapGroupsWithState](https://spark.apache.org/docs/latest/api/java/org/apache/spark/sql/streaming/GroupState.html)
method on a `Dataset` that has been grouped by a key. This is frustrating for a
few reasons. First, what if I don't want to group this operation? There appears
to be no option to skip this step. Second, `groupByKey` is
[explicitly called out as something to avoid](https://databricks.gitbooks.io/databricks-spark-knowledge-base/content/best_practices/prefer_reducebykey_over_groupbykey.html)
in nearly every place it is mentioned online. This is because it performs a
full shuffle of the data. In my case, the grouping was being done by Kafka key,
so it seemed like it would *already be grouped* as it streams from the
consumer.

Structured Streaming appears to assume by default that you are trying to
perform a join operation on topics, and that any grouping being done is
unrelated to how messages are already grouped by topic and partition. I would
argue that this is not a reasonable default. Furthermore, the parallelism model
of your pipeline becomes unclear, because it is no longer necessarily based on
the parallelism present in the Kafka source (until the `groupByKey`, Kafka
partitions map 1:1 with Spark partitions). Sadly this appears to be unavoidable
when doing stateful transformations whilst subscribing to multiple topics, and
in particular when using topic discovery.

#### Latency

One of the main limitations of Structured Streaming that is often
discussed is that it operates on "microbatches". In other words, it is
not strictly speaking a streaming platform but instead periodically processes
data in batches as (mostly) normal Spark queries. This suited me just fine,
as I could adjust the batch interval to achieve the tradeoff between Parquet
file size and latency that I desired. If latency is a critical focus of your
application, I still believe that Structured Streaming is a solid solution with
its
[Continuous](https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html#continuous-processing)
trigger mode. I would also argue that periodically (every ~100ms) processing
new messages in batches is *more* efficient than processing each message
individually as it arrives, because there is some overhead in managing
application state each time progress is made.

#### Final Design

In the end, I wrote the following query in Scala. I packaged this query as a
fat JAR using [sbt assembly](https://github.com/sbt/sbt-assembly), and
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

## Looking Back

Looking back on these decisions, I believe I did a reasonable job of assessing
my options. I have identified a few things that I think I may have overlooked:

- Kafka Streams was clearly the most tightly coupled with the Kafka API
  contract. As we will see in future posts, it may have been a better option
  despite its limitations because of how much I was depending on this behavior.
- Samza has reached `1.0`, but the project has not addressed many of my
  concerns. I believe that their vision for stream processing is very focused,
  which is not a bad thing, but may be incompatible with my needs.
- I believe that I made the right call on Flink. The rapid development over the
  last few years would have been difficult to keep up with. The project has
  improved significantly, and continues to have a bright future.
  Flink has since introduced
  [Stateful Functions](https://flink.apache.org/stateful-functions.html),
  which allow you to piece together an arbitrary graph of stateful actors.
  Between this and improved documentation of Flink's original stateful
  transformation functionality, I would strongly consider selecting it if I
  were starting from scratch today.
- Structured Streaming was technically an immature project at the time that I
  adopted it. It had been around for 3 minor releases, and has subsequently
  proven to be a stable API which needed very few major patches, but at the
  time it was not necessarily the safe choice I estimated it to be.
- My concerns about the grouping operation required by Spark turned out to be
  valid, and I should have spent more time investigating this functionality.
  Interestingly, it appears that Flink and DStreams both also typically perform
  a group by key operation as a prerequisite to stateful transformations,
  raising similar concerns. Flink, however, appears to support a non-grouping
  operation as well, which would be a major focus of mine for future work here.

In the next post I will discuss some of the operational and practical
issues that I've encountered with Spark, and streaming applications in
general. I expect it to be a useful post for those just starting to operate
streaming applications, as well as experienced practitioners looking for
increased performance and reliability.
