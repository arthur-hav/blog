---
title: "Dockerfile for small Rust images (with dependency build caching)"
date: 2023-10-25T12:56:00+02:00
draft: false
---

# Introduction

After reading multiple tutorials for building docker images and optimize them, I compiled an optimized Dockerfile that can:
- Have final images that are small, in the 50MB range
- Benefit from docker caching, allowing to have build times under 10s if you don't change dependencies

We will assume here you start with a project `my_app` you already have or have created with `cargo new`.

# Setting up

We use this docker file. This technique is called a multi-stage build. When docker builds with `docker build` or `docker compose build`, it creates two successive images:
- One that serve to create the binary
- One that will execute the binary without the build environment

The second one can be minimized by removing unecessary system components that are already bundled in the produced binary.


```Dockerfile
## BUILDER IMAGE
FROM rust:1.73 as builder

WORKDIR /usr/app
RUN rustup target add x86_64-unknown-linux-musl
RUN apt update && apt install -y musl-tools musl-dev
RUN update-ca-certificates

WORKDIR /usr/app/api_gateway

RUN adduser \
    --disabled-password \
    --gecos "" \
    --home "/nonexistent" \
    --shell "/sbin/nologin" \
    --no-create-home \
    --uid 10001 \
    userland

COPY ./Cargo.toml ./Cargo.toml
RUN mkdir src && echo "fn main(){}" > ./src/main.rs
# Build the dependencies. This is the longest part and we don´t want
# to repeat it if there is no dependency change,
# which is why there is no copy or volume for sources at this point
RUN cargo build --target x86_64-unknown-linux-musl --release
COPY ./src ./src

# 5. Build for release.
RUN cargo build --target x86_64-unknown-linux-musl --release
## EXECUTOR
FROM alpine
# You can include a healthcheck here, for demons and network-based services.
# For http checks, don't forget curl is not readily available on alpine

# HEALTHCHECK --interval=30s --timeout=30s --start-period=5s --retries=3 CMD [ "curl --fail http://localhost:8000/health" ]
# RUN apk add curl
RUN apk add libc6-compat

WORKDIR /opt
COPY --from=builder /etc/passwd /etc/passwd
COPY --from=builder /etc/group /etc/group
COPY --from=builder /usr/app/my_app/target/x86_64-unknown-linux-musl/release/my_app ./

RUN chown userland:userland ./my_app

CMD ["/usr/app/my_app"]
```

Alpine doesn't natively provide glibc or libssl. For this reason, we will need some additions in the Cargo.toml file.

At end of file, below your dependencies, add the following section:

```ini
[target.'cfg(all(target_env = "musl", target_pointer_width = "64"))'.dependencies.jemallocator]
version = "0.3"
```

You can learn more about what is jemalloc [here](https://jemalloc.net/)

If you use things that are dependent of ssl, such as reqwest, you might also need to tweak the dependency to include `rust-tls`:

```ini
reqwest = {version = "0.11", default-features = false, features = ["json", "rustls-tls"] }
```
## Running

Try to `docker build` your image. If it is successful, try to `docker run` the obtained container. It should execute your binary.

You notice your first `docker build` was probably long, several minutes long perhaps. 

Try to modify your sources, then `docker build` again. You should notice the build to be significantly faster.

Use `docker images` and inspect image size. Smaller images are faster to push and pull over network, they also cost less to store.

# Conclusion

We learned to make a multi-stage build to produce small docker images for Rust, and use the best of docker caching to avoid a costy rebuild
of dependencies at every source change.


## Thanks to / further reading

- [Multi stage build for docker](https://docs.docker.com/build/building/multi-stage/)
- [Notes from cargo team on dependency caching](https://hackmd.io/@kobzol/S17NS71bh)