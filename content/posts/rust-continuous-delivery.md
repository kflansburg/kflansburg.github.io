---
title: "Rust Continuous Delivery"
date: "2020-11-26T00:00:00-00:00"
draft: false
description: In this post I discuss my approach to continuous delivery for Rust
    applications, and how it has evolved over time.
tags:

- AWS
- Terraform
- Rust
- CICD
- Kubernetes

---

Over the last few years I have iterated several times on continuous delivery
pipelines for Rust applications. Designing these pipelines involves balancing
a number of factors including cost, complexity, ergonomics, and rigor. In this
post I will describe several of these iterations, lessons learned, and share my
most recent solution in detail.

<!--more-->

## Requirements for Continuous Delivery

For most teams a continuous delivery (CD) process involves a series of
automated steps, with occasional human gates, which move code from pull request
to production. This ensures that high quality code reaches production, while
reducing the human effort involved in a release.

My use case involved a number of quirks:

- I was the sole maintainer on over a dozen projects. This meant that complex
  `git` workflows did not add much value, but I desired a high level of
  automation and as much help validating the code as possible.
- In rare cases I would need to get a patch into production as quickly as
  possible (10 minutes or less).
- Rust can have long compile times, so incremental builds were important.

## GitLab and Docker Swarm

GitLab was an early mover when it came to integrating CI/CD directly with code
repositories. I was already hosting my code with GitLab at the time, and was
eager to automate the testing and release process. For simplicity, I elected to
use GitLab's container registry for storing images built by the pipeline.

I was also beginning to be overwhelmed with the number services that I was
managing, and looking at container orchestration solutions to integrate with
this pipeline. Like many small ventures, I felt that Kubernetes was too complex
for my needs, and instead opted for Docker Swarm. Despite its limited adoption,
Docker Swarm was a good stepping stone before taking on the complexity of
Kubernetes.

### Cached Rust Container Builds

The most important optimization for Rust build pipelines involves modifying
Dockerfiles to better leverage build caching and produce smaller production
images. This isn't a novel solution, but it is worth mentioning.

First, `cargo build` is split into two steps. The first builds all dependencies
based on `Cargo.toml` and `Cargo.lock`. These layers will only be rebuilt if
Rust releases a new stable version or you modify dependencies. Second, the
application itself is built, this will be rebuilt whenever you modify your
application's source code. Finally, the application's binary is copied to a
second stage which builds the runtime container. It is possible to build an
even slimmer runtime image by statically linking against `musl` and creating
a `FROM scratch` container, but I have not found this to be definitively
better.

```Docker
# Stable
FROM rust:latest as build

# Build Dependancies
WORKDIR /build
RUN USER=root cargo new --bin app
WORKDIR /build/app
COPY Cargo.toml .
COPY Cargo.lock .
RUN cargo build --release

# Build Application
RUN rm src/*.rs
RUN rm ./target/release/deps/app
COPY src/ src/
RUN cargo build --release

# Build Run
FROM debian:stable-slim AS run
COPY --from=build /build/app/target/release/app app
ENTRYPOINT ["./app"]
```

You will need to push both your `build` and `run` images to your registry in
order to properly cache layers. If you only push your `run` image, build
layers will not be included, and will not be available for caching in
subsequent builds. Your build pipeline should first pull previous `build` and
`run` images, then build and tag the new images, and finally push them both.
Since I'm the only user of these images, I typically tag them by commit hash
for precise referencing.

```bash
# Ignore pull failures since this is just for the cache.
docker pull ${REGISTRY_URL}:cache || true
docker pull ${REGISTRY_URL}:latest || true

docker build --target build \
    --cache-from ${REGISTRY_URL}:cache \
    --tag ${REGISTRY_URL}:cache .
docker build --target run \
    --cache-from ${REGISTRY_URL}:latest \
    --cache-from ${REGISTRY_URL}:cache \
    --tag ${REGISTRY_URL}:$(git rev-parse HEAD) \
    --tag ${REGISTRY_URL}:latest .

docker push ${REGISTRY_URL}:cache
docker push ${REGISTRY_URL}:$(git rev-parse HEAD)
docker push ${REGISTRY_URL}:latest
```

### Terraform

After an image is built, I wanted to continue manually triggering the actual
update process. To complete the automation of this deployment pattern,
Terraform was used to provision Docker Swarm nodes and configure workloads in
the cluster itself by connecting to the daemon via mTLS. In this way I could
update the image tag in one place and Terraform would update the many services
using that image. I don't think Terraform is perfect, but I much prefer it to
Helm for composing and validating complex cluster deployments.

## sourcehut then GitHub

Eventually I grew frustrated with GitLab's application performance. I found the
user interface to be fairly inefficient to navigate, and this was coupled with
slow page load times. I felt that the build pipeline on the free tier did not
match other free offerings, and there was not a paid plan which directly
addressed these concerns. A number of minor outages resulted in the inability
to build images for hours at a time every few weeks. Even with caching, build
times were quite long, and I briefly experimented with using GitLab's
self-hosted runner integration. This was quite slick, but was a lot of work to
manage for what was intended to be an integrated solution.

Around this time sourcehut was launched and the lightweight pages with fast
load times were quite appealing. sourcehut also introduced its own integrated
build pipelines, and I felt that even if these were too slow I would be no
worse off than with GitLab. I decided to migrate a few repositories as a pilot.
I determined that sourcehut was great for the reasons described, and is
definitely something I would consider for personal projects, but I quickly
decided that it was too early for production use (which marketing material was
pretty clear about).

These issues were not critical, but I began looking at GitHub again for the
first time since 2016. Microsoft had acquired GitHub, and built out a number of
new features, including Actions, Projects, and Packages. I played around with
some of these features, but the deciding factor was their decision to grant
free accounts unlimited private repositories. At this point, I began migrating
all of my repositories to a new GitHub organization. It is worth noting that
I was still using the same general deployment architecture, just with Actions
and Packages in place of GitLab's offerings.

All of these services are great options, and none of these concerns are deal
breakers. I know that all of these platforms are actively working to address
these concerns. I have clearly spent a lot of time hopping between platforms,
and I would not recommend this unless one of your hobbies is optimizing
developer quality of life.

One thing that I did during this process that I *would* recommend is making
your repositories as uniform as possible. My platform-specific CICD
configurations are exactly the same for every repository of a given language.
This means that migrating between platforms involves updating and testing *one*
script, and then pulling, patching, and pushing can be automated with a Bash
script. If you do need to migrate platforms for some reason, this process can
take just a few hours when combined with Terraform.

## Rust Continuous Integration

I employ a number of steps for validating my Rust code. These are not
revolutionary, but I will include them here for those that are not aware.

1. `cargo fmt --check` Applies `rustfmt`, and fails if there are any changes
   that are expected. This may seem pedantic, but Rust can be hard to parse
   for new Rust programmers, and maintaining a highly idiomatic format in your
   codebase is super easy and can reduce cognitive overhead of reading new
   code.
1. `cargo clippy` applies code-level checks to your application, identifying
   unused imports, non-idiomatic control flow, and unnecessary clones. This
   prevents cruft build up and occasionally improves performance (slightly).
1. The compiler does a lot to detect where type-level API changes break
   consumers of that API. I use unit tests to validate business logic which is
   not caught by the compiler. A common example here is when manually parsing
   binary messages, unit tests can help detect off-by-one errors and other
   subtle bugs that occur when slicing arrays.
1. Rust documentation can include examples which can be run as tests as well to
   validate that they compile and run without error (thus maintaining valid
   documentation). For my use, extensive documentation is not very valuable,
   however these tests can be used to assert that things *don't compile*, which
   can be good for verifying that invalid states are recognized as invalid by
   the compiler. These tests are very brittle, so I use them sparingly.
1. For performance sensitive code, I recommend `criterion` for benchmarking
   critical functions and detecting performance regressions. I currently use
   this only on my local development machine, but it could be integrated into
   a continuous delivery pipeline as well.

These tests will obviously not eliminate bugs, but they can eliminate a lot
of the toil involved in spotting these types of issues, and in doing so ensure
that this is done thoroughly on every commit.

## AWS CodeBuild and Kubernetes

Finally, I would like to share my current pipeline. This evolved over several
months following my move to GitHub. A number of things changed over this time.
First, I had become competent enough with Kubernetes that Docker Swarm began
to feel like the more complex option when compared with Terraform and AWS'
Elastic Kubernetes Service (EKS). The remaining itch I had was build times. In
my previous experiments with GitLab runners, the idea of having control over
the VM size for building was very appealing, but having to either pay for this
machine full time or develop complex behavior for starting and stopping the
machine was daunting.

Eventually I found AWS CodeBuild, which does exactly what I want: trigger
builds with GitHub webhooks, run them on an enormous machine for just a minute
or two, and then shut it down when done. I'm able to achieve insanely fast
builds, and only pay a few cents per build (much cheaper than many CICD
services). Finally, Elastic Container Registry (ECR) was an obvious choice for
container hosting, with simple integration with EKS, default-private
repositories, and immutable image tags. I know this sounds like an AWS ad, but
I expect other cloud providers will catch up eventually if they have not
already. Azure DevOps seems like an obvious candidate for a premium GitHub
Actions option.

This diagram provides a nice overview of my workflow:

![Pipeline](/pipeline.svg)

1. I typically run tests and lints locally, and then push to a development
   branch. I create a pull request for the purposes of code self-review. I then
   merge to `main`.
1. GitHub invokes a webhook to CodeBuild to notify it of the new commit to
   `main`.
1. CodeBuild launches the configured instance type (72 cores, $0.20 / min).
1. The instance pulls code from GitHub using a configured build token.
1. The instance pulls Docker images for caching from ECR.
1. The build script runs rustfmt, clippy, tests, doctests, and then finally
   builds a release binary and Docker image.
1. The runtime and cache images are pushed to ECR.
1. I manually trigger Terraform to update the images specified in Kubernetes
   Deployments.
1. EKS performs a rolling update of my Deployments, pulling the new image from
   ECR.

The icing on the cake, is that *all* of this is configured with Terraform,
including GitHub repo, webhook, CodeBuild job, and ECR repository. Adding a
new codebase is as simple as adding a new entry in a Terraform list variable
and applying the plan.

## Next Steps

I currently still use GitHub Actions to perform validation steps on pull
requests before they are merged. This can still be quite slow and so I do
not require these tests to pass before I merge, they will be repeated during
the release build anyway. I would eventually like to set up a second CodeBuild
pipeline for validation only, and have the results sent back to GitHub for
display in the pull request page.

As mentioned above, I would also like to explore automated benchmarking within
my pipeline, with proper caching of previous results to detect regressions.
This has the advantage that benchmarks will be conducted on a consistently
sized machine with little else running on it. Ideally these results could
be inserted directly into pull requests to document any regressions or
improvements.

## Conclusion

This may seem like a lot of complexity for a one developer operation, but I
think that situations like this are where time spent on DevOps really pays off.
With a little discipline, you can scale quite a bit with even the smallest of
teams. I've identified a couple of key takeaways from this journey:

First, just like servers, codebases for your applications should be cattle, not
pets. The only information you should need for configuring your entire build
pipeline should be the language it is written in and the human readable name
for the app. This eliminates a lot of the complexity of deployment, not only
because you can automate repository operations, but because you no longer
have to remember the idiosyncrasies of each application.

Second, integrated platforms are great, but they all have quirks. You don't
have to jump around, but be aware of what the competition has, it can give
you some great ideas on what you need to improve. If you are willing to
integrate multiple services, you can achieve some really great performance
at low cost, while reducing vendor lock-in.

Finally, infrastructure (and application configuration!) as code
is critical to avoiding toil work. Even if you only plan to set something up
once, I have found the process of developing the code for it can help you to
understand how you are configuring it more thoroughly, as well detect and
revert configuration drift.
