---
title: "Rust Continuous Delivery"
date: "2020-11-26T00:00:00-00:00"
draft: false
tags:

- AWS
- Terraform
- Rust
- CICD
- Kubernetes

---

Over the last few years I have iterated several times on continuous delivery
pipelines for Rust applications. In this post I will describe several of these
iterations, lessons learned, and share my most recent solution in detail.

<!--more-->

# Goals

For most teams a continuous delivery (CD) process involves a series of
automated steps, with occasional human gates, which move code from pull request
to production. This ensures that high quality code reaches production, while
reducing the human effort involved in a release.

My use case involved a number of quirks:

- I was the sole maintainer on over a dozen projects. This meant that complex
  `git` workflows did not add much value, but I desired a high level of
  automation and as much help validating the code as possible.
- In rare cases I would need to get a patch into production as quickly as
  possible. I wanted to target 10 minutes or less with whatever solution I
  developed.
- Rust can have long compile times, so efforts had to be taken to engineer
  around this.

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

### Cached Rust Builds

The first optimization for Rust involves modifying Dockerfiles to better
leverage build caching and produce smaller production images. This isn't a
novel solution, but it is worth mentioning.

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
COPY --from=builder /build/app/target/release/app app
ENTRYPOINT ["./app"]
```

You will need to push both your `build` and `run` images in order to properly
cache layers. If you only push your `run` image, build layers will not be
included, and will not be available for caching in subsequent builds. Your
build pipeline should first pull previous `build` and `run` images, build and
tag the new images, and then push both. Since I'm the only user of these
images, I typically tag them by commit hash for precise referencing.

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
the cluster itself by connecting to the daemon via mTLS. In this way, I could
update the image tag in one place and Terraform would update the many services
using that image. I don't think Terraform is perfect, but I much prefer it to
Helm for composing and validating complex cluster deployments.

## sourcehut then GitHub

I soon grew frustrated with GitLab's application performance. I found the user
interface to be fairly inefficient to navigate, and this was coupled with slow
page load times to create a very aggravating experience. I also grew frustrated
with the build pipeline. A number of minor outages resulted in the inability to
build images for hours at a time. Even with caching, build times were quite
long, and I briefly experimented with using GitLab's self-hosted runner
integration. This was quite slick, but was a lot of work to manage for what
was intended to be an integrated solution.

Around this time, sourcehut was launched, and the lightweight pages with fast
load times were quite appealing. sourcehut also introduced its own integrated
build pipelines, and I felt that even if these were too slow, I would be no
worse off than with GitLab. I decided to migrate a few repositories as a pilot.
sourcehut was quite nice, and is definitely something I would consider for
personal projects, but I quickly decided that it was too early for
production use. A major concern was that the build logs were public, which was
annoying to have to keep in mind when developing the pipeline. Additionally,
at one point the configuration required to build Docker images was changed,
breaking all of my builds. Finally, I encountered issues with using my private
SSH key to authenticate with multiple accounts (business and personal).

These issues were not critical, but I began looking at GitHub again for the
first time since 2016. Microsoft had acquired GitHub, and built out a number of
new features, including Actions, Projects, and Packages. I played around with
some of these features, but the deciding factor was their decision to grant
free accounts unlimited private repositories. At this point, I began migrating
all of my repositories to a new GitHub organization. It is worth noting that
I was still using the same general deployment architecture, just with Actions
and Packages in place of GitLab's offerings.

I'd like to point out that all of these services are great options, and none
of these concerns are deal breakers. I know that all of these platforms are
actively working to address these concerns. I have clearly spent a lot of time
hopping between platforms, and I would not recommend this unless one of your
hobbies is optimizing developer quality of life.

One thing that I developed during this process that I would recommend is making
your repositories as uniform as possible. My platform-specific CICD
configurations are exactly the same for every repository of a given language.
This means that migrating between platforms involves updating and testing *one*
script, and then pulling, patching, and pushing can be automated with a Bash
script. Combined with Terraform, this process can take just a few hours.

## AWS CodeBuild and Kubernetes

Finally, I would like to share my current pipeline. This evolved over several
months following my move to GitHub. A number of things changed over this time.
First, I had become competent enough with Kubernetes that Docker Swarm began
to be the more complex option when compared to Terraform + AWS' Elastic
Kubernetes Service (EKS). The remaining itch I had was build times. In my
previous experiments with GitLab runners, the idea of having control over the
VM size for building was very appealing, but having to either pay for this
machine full time or develop complex behavior for starting and stopping the
machine was daunting.

Eventually I found AWS CodeBuild, which does exactly what I want: trigger
builds with GitHub webhooks, run them on an enormous machine for just a minute
or two, and then shut it down when done. I'm able to achieve insanely fast
builds, and only pay a few cents per build (much cheaper than many CICD
services). Finally, Elastic Container Registry (ECR) was an obvious choice for
container hosting, with simple integration with EKS, default-private
repositories, and immutable image tags. I know this sounds like an AWS ad, but
I expect other cloud providers will catch up eventually. Azure DevOps seems
like an obvious candidate for a premium GitHub Actions option.

This diagram provides a nice overview of my workflow:

![Pipeline](/pipeline.svg)

1. I typically run tests and lints locally, and then push to a development
   branch. I create a pull request for the purposes of self-code review. I then
   merge to `main`.
1. GitHub invokes a webhook to CodeBuild to notify it of the new commit to
   `main`.
1. CodeBuild launches the configured instance type (72 cores, $0.20 / min).
1. The instance pulls code from GitHub using a configured build token.
1. The instance pulls Docker images for caching from ECR.
1. The build script runs rustfmt, clippy, tests, doctests, and then finally
   builds a release binary and Docker image.
1. The image artifact and cache images are pushed to ECR.
1. I manually trigger Terraform to update the images specified in Kubernetes
   Deployments.
1. EKS performs a rolling update of my Deployments, pulling the new image from
   ECR.

The icing on the cake, is that *all* of this is configured with Terraform,
including GitHub repo, webhook, CodeBuild job, and ECR repository. Adding a
new codebase is as simple as adding a new entry in a Terraform list variable
and applying the plan.

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
