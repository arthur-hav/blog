---
title: "Build Rust With Sccache"
date: 2024-05-03T09:11:55+02:00
draft: false
toc: false
images:
tags: 
  - untagged
---

# The problem

In a previous article, I've been discussing optimal ways to rebuild docker images that embed a rust binary. However,
being able to rebuild locally from builder image does not mean you can rebuild it easily in a CI, and that you won't be
wasting time in rebuilding the dependencies over and over.

The tricky part is that within a CI environment, you have to start over from a given image; when the dependencies get
updated, either building that image can be costy, or building from that image can get costy, and there is no way around
that, especially if you intend to work with an automated dependency upgrade tool
such as Dependabot or Renovate, since they will create multiple merge requests for every lib they can upgrade and
rebase it (hence rerunning pipelines) for every commit on the main branch.

## A versatile and cost effective solution

Or is there? Actually, you can cache individual crate compilation result, if they are not dynamically linked with a
system lib; this is possible through an utility called sccache. It require a bit of setup, but we will walk it through;
it's not too complicated.

We will work with AWS S3 buckets. In my experiment, I've been able to save 1:30 of gitlab runner of every compilation.
They are roughly priced 0.01$ every minute on the cloud offer (past free tier), so for 10 complies I "saved" 15 cent
compile, and had 1 cent billed on my AWS, making it cost-effective for the time it saves.

## Setup overview

To setup sccache, I've created a private S3 bucket on AWS, setup an IAM for my CI, and gave it access to the bucket. 
They are located in the interface and there are multiple tutorials you can find for that part. Use a location that
will be in proximity for your CI runners, as this will help reduce latencies.

Then, all you have to do is to setup sccache; I use gitlab variables to transfer credentials in the CI job, that are
located in settings -> CI/CD -> variables, where I defined AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY according to my
IAM creds, RUSTC_WRAPPER to sccache, SCCACHE_BUCKET with bucket name, SCCACHE_REGION with the region, and two other
settings to true: SCCACHE_S3_SERVER_SIDE_ENCRYPTION and SCCACHE_S3_USE_SSL.

We use a Dockerfile and a .gitlabci.yml that will use this. In my setup I build directly in the CI, without DIND.
My Dockerfile is simply

```Dockerfile
FROM rust:latest

RUN cargo install sccache
```

Create a personal access token, then build and push that to registry.

```bash
echo "$MY_TOKEN" | docker login registry.gitlab.com -u username --password-stdin
docker build -t registry.gitlab.com/username/project .
docker push -t registry.gitlab.com/username/project

```

This will create you sccache container image.

Finally, setup the gitlab ci.

```yaml
image: $CI_REGISTRY/username/project:latest

test:
  stage: test
  script:
    - cargo test --release --workspace --verbose

```

It worked on my first try, which is notable and gives me confidence you should be able to reproduce this with minimal
hassle.