---
layout: post
title:  "Kafka Integration Testing with Docker Compose"
date:   2021-02-18 14:30:45 +0000
tags: ["kafka", "testing", "integration testing", "docker", "docker-compose"]
category: kafka
---

# Kafka Integration Testing - Part 1

This is the first part of a series in which I will present a pattern for
integration testing with Apache Kafka using [Burrow][burrowGh] and
[docker-compose][dockerComposeHome]. In this post we will cover how to build a
common docker image that we will then use to run both Kafka and Zookeeper in a
local docker-compose cluster.

The project associated with this series is available on Github
[here][projectGh]. In this post we will cover the contents of the
[first commit][firstCommit].

## Motivations

<!-- TODO: Write this section -->

## Common Dockerfile

We define a [multistage build][dockerMsb] to create our common image. In the
build phase we download and verify the contents of Kafka from the Apache
archive before extracting and installing it. In the following phase we build
the `common` image, that will be the base from which we build our Zookeeper and
Kafka images, by copying over the verified install of Kafka from the `builder`
image and installing the dependencies needed to run it in the docker-compose
environment:

* `openjdk-11-jre-headless` - java runtime environment, needed to run Kafka and
  Zookeeper.
* `wait-for-it` - used to configure docker-compose health checks.
* `ncat` - used to open network ports on our containers to signal they are
  healthy (shown in part 2 when we build the `primer` image).

<!-- TODO: Download the sha512 directly in the container and use to verify. It is -->
<!-- cleaner and with the current implementation this looks fishy from a security -->
<!-- perspective. -->

```Dockerfile
FROM ubuntu:latest as builder

RUN apt-get update && apt-get -y dist-upgrade
RUN apt-get -y --no-install-recommends install \
    curl \
    ca-certificates

WORKDIR /tmp

COPY SHA512SUMS .

RUN curl -fsSL -o kafka_2.13-2.5.1.tgz https://archive.apache.org/dist/kafka/2.5.1/kafka_2.13-2.5.1.tgz

RUN sha512sum --check SHA512SUMS

RUN tar -C /opt -zxf kafka_2.13-2.5.1.tgz


FROM ubuntu:latest

RUN apt-get update && apt-get -y dist-upgrade
RUN apt-get -y --no-install-recommends install \
    openjdk-11-jre-headless \
    wait-for-it \
    ncat && \
    apt-get clean all

COPY --from=builder /opt/kafka_2.13-2.5.1 /opt/kafka

WORKDIR /opt/kafka

CMD trap : TERM INT; sleep infinity & wait
```

## Zookeeper Dockerfile

<!-- TODO: Short adnd high level 'what is Zookeeper' with link to further reading. -->

Our Zookeeper Dockerfile is now very simple, all it needs to do it build from
the `common` image and override the `CMD` to run the script to start Zookeeper
that comes bundled with the Kafka download that we installed, along with the
provided default configuration settings defined in
`config/zookeeper.properties`.

```Dockerfile
FROM common

CMD ["bin/zookeeper-server-start.sh", "config/zookeeper.properties"]
```

## Kafka Dockerfile

Our Kafka Dockerfile is almost as simple as Zookeeper's in that we similarly
build from the common image and start the Kafka server using the bundles script
to do so, however we also copy over a `config` directory containing
`server.properties`, since we need to make a small change to the default
configuration to tell the server where Zookeeper is running.

```Dockerfile
FROM common

COPY config/ ./config/

CMD ["bin/kafka-server-start.sh", "./config/server.properties"]
```

The version of Kafka that we installed in the `common` image contains a copy of
`server.properties` populated with default values, so we can make a local copy
of this that we can edit by building our `common` image, running a container
from it, and copying the file out with `docker cp`:

    $ docker build -t common . # from within the directory with the common Dockerfile
    $ docker run --rm -it --name common common bash
    $ docker cp common:/opt/kafka/config/server.properties . # from another shell outside of the common container

The only change we need to make in this file is to update the value of
`zookeeper.connect` to `zookeeper:2181`, since we will set the hostname of the
Zookeeper container to `zookeeper` in docker-compose.

## Configuring docker-compose

```
version: "2.4"

services:

  common:
    image: common
    build:
      context: common/

  kafka:
    hostname: kafka
    build: broker/
    healthcheck:
      test: ["CMD", "wait-for-it", "--timeout=2", "--host=localhost", "--port=9092"]
      timeout: 2s
      retries: 12
      interval: 5s
      start_period: 10s
    depends_on:
      zookeeper:
        condition: service_healthy

  zookeeper:
    hostname: zookeeper
    build: zookeeper/
    healthcheck:
      test: ["CMD", "wait-for-it", "--timeout=2", "--host=localhost", "--port=2181"]
      timeout: 2s
      retries: 12
      interval: 5s
      start_period: 10s
```


<!-- Links -->
[burrowGh]: https://github.com/linkedin/Burrow
[dockerComposeHome]: https://docs.docker.com/compose/
[dockerMsb]: https://docs.docker.com/develop/develop-images/multistage-build/]
[firstCommit]: https://github.com/rtjfarrimond/kafka-integration-testing/commit/af3a33299e516e4923840d1df98ad0814f9c07ad
[projectGh]: https://github.com/rtjfarrimond/kafka-integration-testing
