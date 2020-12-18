---
title: "Operating Pipelines"
date: 2020-12-05T00:00:02-00:00
draft: true
toc: false
series:

- "Ordered Stateful Streaming"

tags:

- Spark
- Streaming
- Scala

---

In the third post of this series I will share operational and practical issues
that I have encountered when developing and hosting long-running Spark
streaming applications. This includes tips that will be useful to those who are
first starting out with Scala and Spark, as well as insights on performance and
reliability that will be useful to those who are more experienced.

<!--more-->

## Developing Spark Applications

Many readers will know that developing and packaging Spark applications is
often not trivial. I believe the learning curve for this is quite steep, and I
would like to share some issues related to developing and deploying Spark,
Scala, and streaming applications in particular which I have encountered that
proved quite time-consuming to resolve.

### Versioning and JAR Packaging

I personally find dependency management in Spark and Java to be unnecessarily
complex. This is caused by a number of factors:

- Hadoop and Spark have introduced a lot of features in recent releases, and
  documentation often does not indicate what version they were introduced in.
- Many users are operating on vendored clusters which provide older
  versions of these packages. In addition, many vendors provide *forked*
  versions of these packages which do not match the official documentation.
- Issues with dependencies (especially for users submitting from Python) often
  just result in `ClassNotFoundException` or `NoSuchMethodError` at runtime,
  making the debug loop time-consuming and opaque.

The simplest path for specifying custom dependencies, using `--packages` with
`spark-submit`, has many advantages:

- It deals with distributing packages to all nodes.
- It solves for additional dependencies of these packages.
- It avoids cluster-wide version conflicts which may impact other users.

Unfortunately, this does not integrate very well with a modern development
process where a sophisticated package manager handles version resolution for
you. I'd like to share a checklist below that I typically go through when
something isn't working.

#### Clearly Identify Cluster Package Versions

Now that Spark 3 is stable, many vendors are shipping Spark `3.0`. However
Hadoop `2.7` is still often the default. If your cluster predates June 2020,
you will likely find Spark `2.4` or even earlier. Specifying custom Spark
versions using `spark-submit` is unlikely to work reliably (I have not tested
this), however you can generally specify newer Hadoop versions. From here, I
would recommend that you *bookmark* the documentation for your versions in
particular, and be extremely vigilant that any examples you draw from online
are not using newer APIs. I would generally recommend that you always use the
latest Hadoop when possible. Hadoop tends to be the biggest source of "missing"
features for me, and there are massive performance improvements in Hadoop
`3.x`.

#### Audit Specified Dependencies

Double check the dependencies that you are specifying for `spark-submit` to
ensure that the versions that you have selected for each package do not cause
conflicts. One common error when using Scala is specifying the wrong Scala
version of these packages. You should also check that you are using a Scala
version matching the cluster. While it can be customized, Spark `2.4.2+` will
be Scala `2.12`, and anything before that will be Scala `2.11`. It is possible
that dependencies have conflicting sub-dependency versions. I typically spend a
lot of time on [Maven Repository](https://mvnrepository.com/) investigating
dependencies during this stage. For streaming pipelines that interact with
Kafka, you will need `spark-sql-kafka-0-10`, which is not bundled by default.

#### Packaging "Fat" JARs

One option for automating this process is to let your local package manager
and compiler do this dependency resolution and validation of existing APIs for
you. To ensure that these validated dependencies are available on the cluster,
you can package them along with your application in what is called a "Fat" JAR.
In Scala, the `assembly` plugin for `sbt` appears to be the most popular way
of doing this.

This process is not straightforward at times. Packaging these files together
requires a process for resolving conflicting filenames. The `assembly` plugin
provides a `case`-based approach to resolve these conflicts, along with a
default implementation. Unfortunately, you almost always have to manually
resolve some files. Many of these files are not used at runtime and can be
discarded, however some require some deeper investigation.

When I first attempted this, I was greeted with *thousands* of conflicting
files, and spent quite a bit of time trying to resolve them. This was due to
a misconfiguration of `build.sbt`. It is **extremely critical** that you mark
dependencies which the cluster will provide (Spark for example) as `provided`
in this file, or else `sbt` will try to package *everything* into your JAR and
you will encounter a lot of conflicts. With this change, I still had a handful
of conflicts, but it was a much less daunting process.  Note the `provided` to
indicate not to include a package:

```scala
libraryDependencies += "org.apache.spark" %% "spark-sql" % "3.0.1" % "provided"
```

## Memory Usage in Streaming Applications

The resource usage of streaming applications is very important. Many Spark
queries run for only a few minutes, but streaming applications may consume
cluster resources for months on end. The ability to use one less server for
your query can translate to thousands of dollars in savings.

Due to the Java Virtual Machine (JVM), Spark has a well-deserved reputation for
poor memory management. The key here is that users have to take on the role of
the operating system and carefully specify configuration parameters so that
they can fully utilize the hardware that they are running on. This process is
extremely time-consuming, and even the most sophisticated vendors get this
wrong, or are not optimizing for streaming queries.

In addition to this, stateful streaming in particular appears to struggle with
memory issues. There are a number of contributing factors here:

- While Spark appears to be memory-efficient when loading Kafka data, the
  entire state of a stateful transformation must be kept in memory during a
  batch. Spark appears to keep states for all groups assigned to a given node
  in memory simultaneously. While this is typically not an enormous data
  structure, it is something to keep in mind.
- By default, Kafka streaming queries will attempt to consume all new offsets
  in a single batch. If your application is starting up after some downtime,
  this could mean millions of records. You should specify
  `maxOffsetsPerTrigger` to limit this behavior, but be careful that your
  application can still keep up (or catch up) with the data in Kafka.
- As mentioned earlier, stateful transformations require the `groupByKey`
  operation which causes a full shuffle. Combined with the previous issue,
  this can exceed available memory.

What I can recommend here is that you treat tuning of these parameters as a
process that begins when you first deploy your application. Here are some
things to keep in mind:

- Tools like Ganglia can provide a lot of insight into resource usage of your
  query.
- I recommend that you err on the side of too much memory when first starting
  out, as out-of-memory crashes can have undesirable results.
- If the throughput of your input data varies over time, make sure that you
  monitor resource usage over this full period to understand peak usage.
- It may be a good idea to start the pipeline with several days worth of
  unprocessed records to get an idea of resource usage during a recovery
  situation.
- Reduce memory configuration slowly, and remember that both Spark and the JVM
  have their own settings.
- Once you have identified reasonable settings for your application, make sure
  to keep an eye on resource usage if the throughput of your input data grows
  over time.

## Recovering from Failures

Due to numerous out-of-memory crashes, I have had plenty of opportunities to
stress test the recovery feature. In general, it seems to work as advertised
but it feels like a weak point given the consequences of failure (multi-hour
gaps in data).

The first practical consideration is Kafka topic retention. The real trade off
here is between disk space cost and the time to respond to application
failures. When the processing pipeline goes down, your are now on the clock to
fix it before you start losing unprocessed records. I would recommend an
*absolute minimum* of 72 hours retention to accommodate weekends. If
engineering determines that a job cannot be resumed right away, it may make
sense to temporarily increase the retention of affected topics. Keep in mind
that if you are sizing your block volumes based on this retention and cost
savings, temporarily expanding a Kafka storage volume may not be
straightforward. An important factor here becomes reliably notifying a human
when the job fails. Unfortunately this can be difficult to integrate with
Spark, and most of my solutions have involved terrible Bash scripts.

Spark's recovery mechanism is not 100% bulletproof, and when it does fail you
tend to find yourself in a pickle because the correctness guarantees in the
application become your enemy when checkpoint data and output data no longer
agree. Structured Streaming makes it clear that S3 is not a valid location to
store checkpoint data, because it does not provide the consistency guarantees
that a real filesystem does. What I have found, though, is that Spark appears
to still insist on storing some metadata in S3 and uses this in combination
with the actual checkpoint to recover the job, preventing you from fully
avoiding these consistency issues. In particular, Spark appears to look at
"batch" numbers within the S3 metadata and skip batches which have already
"occurred", which semantically makes no sense because batch numbers are
meaningless and will not contain the same offsets from job to job. Luckily
this appears to be rare, and I've managed to mostly avoid it, but I consider it
to be a bug.

A general process that I've developed for repairing issues like this is:

1. Include offsets in output data.
1. When a failure occurs, identify the offset of the last state-of-the-world
   snapshot message for each Kafka partition.
1. Use a separate Spark job to delete output data which comes after these
   offsets, to avoid having duplicate data.
1. Manually specify these starting offsets for your streaming job.
1. Back up and then delete checkpoint data (both HDFS and S3) for the streaming
   job.
1. Start your new streaming session.

## Migrating from Vendored Solution to Kubernetes

Vendored Spark solutions may not be the best choice for pure Spark streaming
queries like this. As mentioned earlier, some vendors offer customized versions
of packages, which can offer significant performance improvements and great new
features but complicates the development process and introduces lock-in. Most
vendors do offer the latest versions of packages, but deeper levels of
customization can be made more difficult by managed offerings.

In my experience with vendored solutions, I have noticed several undesirable
things. First, many ([not all](https://cloud.google.com/dataproc)) run a full
Yarn managed Hadoop cluster. For pure Spark applications this is a lot of
overhead and I found myself needing nodes almost twice the size of the nodes I
used for full-scale testing. Second, many vendors tend to run their Spark
clusters "hot", configuring memory settings higher relative to the hardware
available, preferring less swapping of data to disk in exchange for more
frequent lost tasks. For a traditional Spark job, this makes a lot of sense.
Spark handles re-execution of the occasional lost task gracefully. For
streaming jobs, however, it is much less desirable to occasionally lose
executors, and I have found this to be one of the largest causes of failures
for which smooth recovery is not possible.

Eventually I migrated this workload to Kubernetes. Self-managed Spark
clusters on Kubernetes are actually quite doable, and offer a number of
advantages:

- Container images can be built with the exact dependencies that your job
  needs, simplifying the continuous delivery process.
- Clusters can be dynamically provisioned for single-tenant workloads using
  [one of the many](https://github.com/bitnami/charts/tree/master/bitnami/spark)
  Helm charts out there.
- Kubernetes appears to have lower overhead, and I have found that resource
  usage is much more predictable.
- Running Spark in standalone mode is perfectly fine, and reduces complexity.
- Streaming queries can be submitted in client mode as a `Job` for
  Kubernetes-native tracking of application failures and retries.

If you are an organization which already leverages Kubernetes, I definitely
recommend exploring this approach. If not, similar results can be achieved with
AMIs and Terraform to automate provisioning of single tenant Spark standalone
clusters. If you go with either of these routes, I definitely recommend
installing a log aggregation solution for quickly investigating issues, as
digging through Spark logs in the Web UI or on the nodes themselves can be
very cumbersome.

## Conclusion

I hope that this post contains some useful and time-saving tips.  Many
organizations will already have a lot of best practices for deploying Spark and
Hadoop applications, but may still benefit from considering how streaming
applications have different requirements than batch jobs. For those new
to Spark, this may seem daunting, but I have tried to include all of the major
speedbumps that I have encountered, which should save you a lot of time.

In the next post I will analyze the various types of failures that may occur
in a streaming application, and discuss possible mitigations to reduce or
eliminate data loss.
