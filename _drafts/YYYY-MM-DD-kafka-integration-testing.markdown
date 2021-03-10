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
the `common` image that will be the base from which we build our Zookeeper and
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

Our Zookeeper Dockerfile is now very simple, all it needs to do is build from
the `common` image and override the `CMD` to run the script that starts
Zookeeper that comes bundled with the Kafka download that we installed, along
with the provided default configuration settings defined in
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

    # from within the directory with the common Dockerfile
    $ docker build -t common .
    $ docker run --rm -it --name common common bash

    # from another shell outside of the common container
    $ mkdir -p config
    $ docker cp common:/opt/kafka/config/server.properties ./config/server.properties

The only change we need to make in this file is to update the value of
`zookeeper.connect` to `zookeeper:2181`, since we will set the hostname of the
Zookeeper container to `zookeeper` in docker-compose.

## Configuring docker-compose

We define `common`, `kafka`, and `zookeeper` services in our
`docker-compose.yml` file. Although we do not need a running instance of the
`common` container for our tests, it is still specified here since we want
`docker-compose build` to build the image, since it is the shared base image of
the other two services.

Kafka depends on Zookeeper for orchestration of broker nodes in the distributed
system. Even though we have only one broker server in our example the
dependency still exists, so we make it explicit to docker-compose using
`depends_on` in conjunction with `service_healthy` and `healthcheck`. It is
worth noting that it is possible to configure docker-compose to run multiple
broker servers by adjusting [the `scale` parameter][composeScale], however we
will not do so in this example.

The health checks use the `wait-for-it` package that we installed in the
`common` Dockerfile, and will check whether a port is open on the localhost. In
practice this means that when the `zookeeper` service is running,
docker-compose should consider it to be in a healthy state as long as port 2181
is open, after a startup grace-period of 10 seconds. Once `zookeeper` passes
its first health check, docker-compose will then start the `kafka` service.

We are now ready to run our local Kafka set up by running `docker-compose up
--build`! You should see from the logs that both services start up without
issue. When you are done, run `docker-compose down` to clean up the containers.

If we had not overridden the default configuration for the Kafka server to
specify where Zookeeper is running we would have observed connectivity issues
when the services started. Go ahead a change the value of `zookeeper.connect`
to some other value and rebuild the images. You should see that when you run
`docker-compose up` again after the rebuild Kafka now complains in the logs
that it cannot connect to Zookeeper before eventually exiting with `1`.

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

## Still to Come

In the next part of this series we will introduce [Burrow][burrowGh] and use it
to run a very simple test. We will configure another service in docker-compose
whose responsibility it will be to create a topic, produce a known quantity of
messages to the topic, and consume a known quantity of messages from the topic.
Burrow will be used to verify that production and consumption both occurred as
expected. This will be our first test that really illustrated the pattern of
how we will run our tests in this configuration.

In part 3 and beyond we will look at how to create simple producers and
consumers using Scala and [fs2-kafka][fs2KafkaHome], and how to test these with
the docker-compose pattern. It is worth noting however that the choice of
language and framework really are not important, so long as your producers and
consumers run inside docker containers you can use the pattern presented here.

<!-- Links -->
[burrowGh]: https://github.com/linkedin/Burrow
[composeScale]: https://docs.docker.com/compose/compose-file/compose-file-v2/#scale
[dockerComposeHome]: https://docs.docker.com/compose/
[dockerMsb]: https://docs.docker.com/develop/develop-images/multistage-build/]
[firstCommit]: https://github.com/rtjfarrimond/kafka-integration-testing/commit/af3a33299e516e4923840d1df98ad0814f9c07ad
[fs2KafkaHome]: https://fd4s.github.io/fs2-kafka://fd4s.github.io/fs2-kafka/
[projectGh]: https://github.com/rtjfarrimond/kafka-integration-testing
