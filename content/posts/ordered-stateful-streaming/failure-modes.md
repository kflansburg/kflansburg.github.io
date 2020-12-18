---
title: "Failure Modes"
date: 2020-12-05T00:00:03-00:00
draft: true
toc: false
series:

- "Ordered Stateful Streaming"

tags:

- Spark
- Streaming
- Scala

---

In the fourth post of this series I will outline several sources of failure in
streaming applications. In each case I discuss the potential for data loss and
possible mitigations. This analysis can drive decisions on the tradeoff between
infrastructure costs and the upper bound on data loss.

<!--more-->

## Failure Modes and Mitigations

Many treatments of streaming applications focus on mechanisms that frameworks
use to avoid or gracefully recover from failures. My experience, however, is
that they cannot operate perfectly forever, and your organization cannot rely
on exactly perfect data. A more practical approach is to identify possible
situations which could lead to data loss, and make plans to limit this data
loss and recover the application. This also provides an opportunity to
reach an agreement with data consumers regarding the level of service and the
cost implications thereof. In this post I will outline the major sources of
failure that I have encountered, in order of increasing severity.

## Environmental Failures

I define environmental failures as anything that causes a task failure that is
unrelated to the data being processed or the job code itself. My memory issues
are an example of this, where settings can simply be adjusted and the task
restarted. Another example might be a Pod eviction on Kubernetes triggered by
the cluster autoscaler. The insight here is that these are in general the only
types of failures which snapshot and recovery saves us from.

## Data Integrity Failures

Data integrity errors include malformed checkpoints or malformed Kafka topic
contents (typically caused by a bug in the producer) which prevent the
streaming job from continuing. In my experience these errors result in gaps in
data. Critically, the gap will be from the last valid message until the next
state-of-the-world snapshot. Pipelines with strict SLAs should therefore send
full snapshots at a fixed interval to establish an upper bound on data loss.
Depending on the size of your session state, this could be as frequently as one
to five minutes.

## Kafka Data Loss

Loss of Kafka data due to a failure in its underlying storage can result in a
full loss of unprocessed partition data. If your pipeline is running and caught
up when this happens, you could in theory lose just a few minutes of data. In
practice, however, it will take several hours to manually recover. This process
involves recovering the Kafka cluster and then following the process described
in the previous post for removing checkpoints and restarting your job from an
offset. Things get more complicated when only some of your partitions are lost,
because you will need to identify which ones, and only remove those
checkpoints. This failure mode is definitely a weak point, and I would
recommend using multi-zone topic replication to avoid it as much as possible.
Additionally, scripting and testing this recovery process can significantly
reduce downtime in the event that such a failure cannot be avoided.

## Full Reprocessing of Historical Data

There are a few failures that could require you to reprocess the full topic
history from Kafka. A simple example would be loss of output data due to a
failure in object storage. This is extremely unlikely, however, and can be
mitigated with cross-regional replication. Much more likely would be a bug in
the business logic of your stream processing job which invalidates your output
data. This is usually a disaster, because most organizations do not want to
retain Kafka records forever, and even if they did, reprocessing that many
records will be time-consuming and expensive. I recommend a number of
considerations here:

- When designing the system, determine if there is an obvious upper limit to
  the age of individual partitions, perhaps just a few weeks or months. This
  could make for an easy win in selecting a retention policy.
- Agree on an upper limit SLA for data retention in Kafka. This way, everyone
  is aware of how much history will be retained in the event of such a loss,
  and the cost implications of this retention.
- Be reasonable about the degree of error in your output data. In the best
  case, the historical data may be able to be repaired with an ad-hoc ETL job.
  In others, it may be still possible to use the data for some purposes.
- There are a number of mechanisms for rolling Kafka data to cheaper object
  storage for long term retention. I have not had an opportunity to test this,
  and it introduces a great deal of additional complexity, but may be worth a
  look.

## Conclusion

This analysis of potential failures and their mitigations is not exhaustive,
but should provide a good process for preparing to deal with eventual issues.
For more complex pipelines, you will need to perform a similar analysis on
other inputs to the pipeline to identify and plan for situations where that
data is unable to be processed with integrity.

In the next post I will return to the discussion of my application in
particular to describe my debugging approach for dealing with the error
mentioned at the beginning of this series.
