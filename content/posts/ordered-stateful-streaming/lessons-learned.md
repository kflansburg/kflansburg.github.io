---
title: "Lessions Learned"
date: 2020-12-05T00:00:05-00:00
draft: true
toc: false
series:

- "Ordered Stateful Streaming"

tags:

- Spark
- Streaming
- Scala

---

In the final post of this series I will summarize what I think are important
lessons that can be learned from my experiences with developing and operating
a stateful streaming application over the last few years.

<!--more-->

## Validate Your Requirements

Because stream processing is still a rapidly changing technology, it is
important that you take the time to understand and be clear about what
functionality is required in order to correctly perform the types of streaming
queries that you are developing. Take the time to dig into each framework that
you are considering to ensure that it supports these requirements. It can be
very easy to optimistically project your needs onto new technologies that you
are considering.

In my case, Spark's Structured Streaming documentation addressed a lot of my
requirements directly, and in greater detail than any other tool. This
contributed to an assumption that everything would operate as I believed it
should. When considering a large number of frameworks, it is not practical to
become a complete expert on each, you must often rely on examples and
high-level documentation to make your decision. It was easy to look at their
concise examples, in the context of my knowledge of the Kafka API contract, and
assume the correct behavior.

I believe that I was doing the right research by confirming the behavior of
each framework surrounding data integrity, but in the future I will be more
methodical about confirming any assumptions made on the first pass. I believe
if I had done this at the time, Structured Streaming would have come in second
place to Kafka Streams, and I would have decided that Streams' limitations
were more tractable to work around than Spark's.

## Understand Your Stack

For sufficiently complex use cases or large enough scale, you will end up
managing the entire stack. It is easy to pick off-the-shelf tools like Spark
or Kafka, with a variety of managed offerings, and still end up needing to
understand them in low-level detail. This may include tuning settings, using
non-standard dependencies, and even reading source code.

I don't think this is a bad thing. These are complex technologies, and this is
an emerging use case. It is not reasonable to expect that a single tool or
service will abstract over all of the complexities. You can still start with
managed offerings and migrate towards self-managed as you learn more. The key
here is to be aware of the commitment you are making when you decide to use
new tools and to do as much homework up front as necessary to understand that
commitment.

## Prepare for Failure

A lot of my investigation into these frameworks was spent on understanding
their recovery capabilities. It took me several months to realize that it is
simply not practical to have a "perfect" pipeline with no gaps in output data.
When you are investigating features that are intended to improve the
reliability of your applications, it is important to look at *their*
dependencies, and understand how your application can still fail. This planning
will reduce surprises and allow you to limit the impact of eventual failures.

## Improve Your Testing

Debugging streaming applications is very complex, and the exercise of building
test harnesses for such systems is very much left to the reader. There are a
number of high-value components to a good testing system for these
applications:

- Perform validation in your pipeline at every step of the way. Debugging
  errors downstream is much more time-consuming.
- Include any metadata you can in the output. Things like Kafka partitions and
  offsets may seem extraneous in the output data, but in my experience they add
  very important context to each row.
- Have tools (often just Python scripts) for validating business logic at
  every step of the pipeline, including: raw input data, data in Kafka, output
  data.
- Be able to run the pipeline in a tracing mode to collect additional
  information about its internal state. This will involve building tooling
  for log collection and parsing.
- Optimize the debugging loop as much as possible. This involves one-click
  containerized deployment, incremental builds, and precise control over
  parameters such as trigger interval and what subset of the data to process.

I am very much a believer in strong DevOps and pervasive
infrastructure-as-code, and this application is no different. With so many
moving parts it is important that you can automate as much as possible the toil
involved in standing up test pipelines, managing your production pipeline, and
monitoring the integrity and health of your data.

## Wrap Up

I hope that this has been an interesting series. I believe that stream
processing is an exciting field of computing, and I plan to continue exploring
the many frameworks and paradigms that have been developed to accomplish
stateful transformations on streaming data. This experience has shown me that
a lot of work has gone into solving the complexities of these systems, but also
that a lot of work remains to be done. I hope I'm able to take some of these
lessons and contribute to that effort.
