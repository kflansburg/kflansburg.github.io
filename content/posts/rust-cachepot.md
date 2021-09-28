---
title: "Speed up Rust Builds with Cachepot"
date: "2021-09-27T00:00:00-00:00"
draft: false
description: An introduction to using Cachepot, a derivative of sccache, to
  cache Rust build artifacts and improve build times.
toc: false
tags:

- Rust
- CICD

---

One of the most effective ways for speeding up Rust builds is to cache the
compiled artifacts of crate dependencies. Cargo does this automatically for
local builds, but this quickly breaks down for distributed scenarios.

In this post, I will share my experiences with configuring and using Cachepot,
a tool which wraps the Rust compiler and automatically caches build artifacts
using a variety of cloud storage options. This creates a cache which can be
shared amongst teams, used in ephemeral CI/CD environments, and even used for
distributed builds.  

<!--more-->

## Cargo Cache

When building Rust crates locally, the simplest option is to use Cargo's
built-in caching functionality. In some cases this approach can also be used in
CI/CD pipelines. Many CI/CD tools allow you to manually specify cache paths, and
preserve their contents across builds. A few tools are able to configure this
automatically, such as the
[Rust Cache](https://github.com/marketplace/actions/rust-cache) action for
GitHub Actions.

Unfortunately Cargo's local cache is not intended to be used in a distributed
fashion. Additionally, depending on how your CI/CD tool persists these files, it
may not be possible to use them in parallel builds. These limitations likely
drove Mozilla to begin work on `sccache` in late 2016. 

## sccache

`sccache` was designed to be similar to `ccache`, but with support for
Rust and cloud storage backends. At the time of this writing, `sccache` has
become a mature project, and appears to be fairly well known, but is somewhat
notoriously difficult to configure. Recently, the pace of development appears to
have slowed, and a number of critical updates have not been accepted.

One important update is the support of Amazon's
[signature version 4](https://docs.aws.amazon.com/general/latest/gr/signature-version-4.html)
for authenticating requests to private S3 buckets. Regions built after 2013 only
support version 4, and Amazon
[stopped supporting version 2](https://aws.amazon.com/blogs/aws/amazon-s3-update-sigv2-deprecation-period-extended-modified/)
in the remaining regions for buckets created after June 24, 2020. As a result,
it is impossible to use `sccache` with buckets created today. Several PRs have
attempted to fix this issue by switching to the `rusoto` crate, however
[the latest](https://github.com/mozilla/sccache/pull/869)
was blocked in favor of waiting for the
[new official AWS SDK](https://aws.amazon.com/blogs/developer/a-new-aws-sdk-for-rust-alpha-launch/)
to stabilize.


## Cachepot

While `sccache` is still considered to be actively developed,
[Parity Technologies](https://www.parity.io/) has forked the project under the
name Cachepot. This effort appears to have started around April 30, 2021 and be
lead by Igor Matuszewski ([@Xanewok](https://github.com/Xanewok)) and Bernhard
Schuster ([@drahnr](https://github.com/drahnr)). In addition to the S3 patch
mentioned above, Parity says that Cachepot includes "improved security
properties and improvements all-around the code base", which they share upstream
when possible. Given the impasse I had reached with `sccache`, I decided to
give Cachepot a try. For simplicity, I will refer only to Cachepot, but many of
the features I will describe here are `sccache` features which Cachepot
inherited.

### Install Cachepot

Installing Cachepot can be done easily using Cargo: 

```bash
 cargo install --git https://github.com/paritytech/cachepot
```

For use in a CI/CD pipeline, it is preferable to install pre-compiled binaries,
however the repository does not currently include these in releases. This
appears to be [coming soon](https://github.com/paritytech/cachepot/issues/101),
but for now I had to install locally and then upload the resulting binary to
S3 for use in my pipeline.

### AWS Infrastructure

Cachepot supports a number of backends, including AWS, GCP, and Azure object
storage, Redis and Memcached, as well as local storage. Since I use AWS
CodeBuild, an S3 bucket in the same region seemed to be the best choice. 

1. Create an S3 bucket in the region you are building in.
1. Create a new IAM user and access credentials for your build jobs.
1. Create an IAM policy for your user, which grants access to the bucket:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Resource": [
                "arn:aws:s3:::<YOUR_BUCKET_NAME>",
                "arn:aws:s3:::<YOUR_BUCKET_NAME>/*"
            ],
            "Action": [
                "s3:*"
            ]
        }
    ]
}
```

**Note:** If you are using `docker build` inside of CodeBuild, it may be tempting
to use the IAM role attached to your build job. This
[can be done](https://blog.jwr.io/aws/codebuild/container/iam/role/2019/05/30/iam-role-inside-container-inside-aws-codebuild.html)
, but you must pass the `AWS_CONTAINER_CREDENTIALS_RELATIVE_URI` environment
variable as a Docker build argument. _Unfortunately, this completely invalidates
your Docker cache!_ For this reason, I chose to use static credentials.

### Using Cachepot

Now that you have Cachepot installed and storage provisioned, you can use it by
setting just a few environment variables:

```bash
#!/bin/bash
RUSTC_WRAPPER=<PATH_TO_CACHEPOT_BINARY>
CACHEPOT_BUCKET=<YOUR_BUCKET_NAME>
AWS_REGION=...
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...

cargo build --release
```

That's it! I believe a lot of people (myself included) are thrown off by parts
of the documentation related to starting the Cachepot server. This is not
necessary (especially in CI), because Cachepot will automatically start the
server if it is not already running. 

You can verify that Cachepot is running most easily by checking that files have
been written to your S3 bucket. 

### Debugging

If you are having trouble getting Cachepot to work, you can enable debug logging
by setting the environment variable `CACHEPOT_LOG=debug`. This is the best you 
can do in CI, but it does not paint the full picture because the most important
logs (such as S3 authentication errors) will be from the server which is running
as a daemon. To view these, you can build locally, with Cachepot server running
in the foreground:

```bash
#!/bin/bash
# Start Cachepot Server
# Note that this command needs the S3 config, not the client.
CACHEPOT_BUCKET=<YOUR_BUCKET_NAME>
AWS_REGION=...
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
CACHEPOT_LOG=debug
CACHEPOT_START_SERVER=1
CACHEPOT_NO_DAEMON=1

cachepot
```

```bash
#!/bin/bash
# In a separate terminal, build your project
RUSTC_WRAPPER=<PATH_TO_CACHEPOT_BINARY>
cargo build --release
```

### Caveats

There are a few things to keep in mind when using Cachepot:

* The absolute path of the build directory must match for cache hits to occur.
  If you wish to share a cache between local and CI, or across developer
  machines, be sure to use the same absolute path in all cases. 
* Cachepot cannot cache "linked" crates, such as "bin" and "proc-macro"
  crates. Dependencies will never include binary crates, but might include some
  proc-macro crates. 
* Cachepot does not support incremental compilation. In my experience, this is
  not an issue because the primary goal is to cache dependencies, which are
  (almost) never compiled incrementally.
* You may want to disable the use of debug information (`-C debuginfo=0`), to
  reduce the size of the cached artifacts, and reduce upload/download time.

Note that incremental compilation and debug information are already disabled
for the `--release` profile, so if you are only building a release binary in CI,
then you may not have to make any changes here!

## Summary

Cachepot (or `sccache`) can seem daunting to set up at first, but offers
significant performance improvements in ephemeral build environments. For even
more caching goodness, I was able to use Cachepot with
[cargo chef](https://github.com/LukeMathWalker/cargo-chef), right out of the
box! I have now been using Cachepot for several weeks, and have not run
into a single issue, with a reduction in average build times of 60%!

For persistent build servers (with plenty of memory), the Redis or Memcached
backends can offer an even greater boost in performance. Finally, if you are
interested in distributed builds, check out the
[distributed quickstart guide](https://github.com/paritytech/cachepot/blob/master/docs/book/src/dist/quickstart.md)
, which seems ripe for a follow-on post involving Kubernetes!