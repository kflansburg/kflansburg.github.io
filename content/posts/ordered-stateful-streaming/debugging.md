---
title: "Debugging Process"
date: 2020-12-05T00:00:04-00:00
draft: true
toc: false
series:

- "Ordered Stateful Streaming"

tags:

- Spark
- Streaming
- Scala

---

In the fifth post in this series I will describe the approach that I used to
systematically track down a bug in my streaming application. This process can
be very complex and time-consuming for distributed pipelines such as mine. By
reviewing what was required to identify the bug, we can identify opportunities
to improve the tooling used to develop and debug streaming applications.

<!--more-->

## Symptoms

After a few months of operation, I began to notice errors in the output data
that must have been caused by a loss of data integrity. It was clear that rows
were getting deleted then inserted (rather than inserted then deleted),
resulting in stuck rows. This was the most obvious symptom, but it called into
question the validity of the entire table. Unfortunately this bug was only
presenting for certain capture sources, and occurred less than once a day,
making it difficult to reproduce and investigate.

There are many components that must be validated when debugging applications
such as this. While observability techniques like resource monitoring, log
aggregation, and tracing can help in these situations, the level of
granularity needed to track down many bugs is not practical for production use.
This results in a very complex testing setup and unfortunately a very long
debugging feedback loop.

## Data Source and Producer

Given that the issue appeared to only be present for certain data sources, I
began investigating the messages being produced to Kafka. The data sources that
I collect from are complex in that they produce stateful streams themselves,
which do not include sequence numbers, and occasionally lack integrity. Because
of this, my capture application must actually maintain its own copy of session
state so that it can perform integrity checks and start a new session if an
error is detected. After a detailed audit of this code, I determined that there
was a bug that would prevent a session restart when integrity was lost.

I wanted to verify that this bug was the culprit, so I made some modifications
to the application to log raw messages from the data source, and ran it without
the bug fix to confirm that it was failing to restart on loss of integrity.
After several days, I noticed the error in the output data of the
pipeline. From the capture application's logs, I could see that it had not
restarted the capture session. I then ran a Python script to independently
validate the messages from the data source, and found that they were in fact
valid and there was no loss of integrity. It would seem that, while it was
good to discover this bug, it was not causing the errors in the output of
the pipeline.

## Kafka Data

Next I turned my attention to Kafka. Since I had collected a topic which
would result in invalid output data, I removed the other topics and set
retention to forever so that I could focus on the data in question. I wrote a
second Python script to validate messages stored in this topic, and determined
that they were valid as well. This narrowed my focus to the Spark Streaming
application.

## Stateful Streaming Application

The process of debugging a Spark Streaming application can be very arduous.
This is caused by a number of factors:

- The internal state of the application can be extremely opaque. It is
  undesirable to log output for every single message it processes, and
  logging for Spark applications is not very straightforward.
- The feedback loop for debugging is extremely long, requiring an `sbt` build,
  followed by several minutes waiting for the application to start and process
  a batch, and finally sifting through Spark logs and Parquet files.
- When you do want to do "println debugging", the distributed nature of Spark
  makes finding and reading this output tricky.

I ended up developing a lot of tooling here, including Bash scripts, Dockerized
testing, and a Python script to read both Kafka and Parquet and identify
discrepancies. After a number of days, I determined that the `Iterator`
supplied to my stateful function *was not ordered within batches*. It appears
that this is triggered by processing large batches which (presumably) exceed
Spark's max partition size, causing the sporadic behavior of the problem.

This behavior is not obvious from Spark's documentation for several reasons:

- It is not mentioned in the Structured Streaming [documentation](https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html#arbitrary-stateful-operations).
- For small inputs, it is ordered, preventing discovery of this behavior during
  development.
- `flatMapGroupsWithState` appears to behave differently for streaming and
  non-streaming `Datasets`.
- It is not clear how this happens: Spark appears to lazily iterate Kafka
  messages, so either it is opting to create two consumers for the same
  partition, or it is loading all of the messages into a single `Dataset` and
  applying Spark's generic `groupByKey`.
- The actual documentation of the behavior is buried [here](https://spark.apache.org/docs/latest/api/scala/org/apache/spark/sql/streaming/GroupState.html).

Spark appears to assume that you want to mingle records between topics, and
that grouping is not happening by partition. Even if you only subscribe to a
single topic, the grouping operation will discard the ordering of the messages.
Given that `flatMapGroupsWithState` is the only mechanism for applying stateful
transformations, the implication here is that Spark *does not support ordered
stateful transformations* in a way that respects the semantics of the Kafka
API. As mentioned in the second post, other frameworks leverage a grouping
operation as well, before applying stateful transformations, making me wary of
moving forward with any of these frameworks until further research can be done.

## Mitigations

Unfortunately this was a pretty damning discovery, and would mean that I needed
to migrate away from Spark for this application. As a temporary mitigation, I
experimented with some solutions. The first was to configure
`maxOffsetsPerTrigger` and `spark.sql.shuffle.partitions` to try to force
Spark to process the messages as a single partition. This does not feel like a
great solution because it is unreliable and will deeply impact performance.
Regardless, it did not appear to fix the problem, at least for throughputs that
would still allow my application to keep up with the data in Kafka.

Finally, I modified my stateful transformation to keep track of the offsets
that it had seen and push out-of-order messages onto a `PriorityQueue`,
replaying them back at the end of the batch. This significantly increases the
memory footprint and processing time of a batch, and handicaps the throughput
of the application. Unfortunately it is the best stopgap that I have found
while I develop a replacement application.

In the next and final post of the series I will reflect on what I have learned
from this project, and the debugging process in particular.
