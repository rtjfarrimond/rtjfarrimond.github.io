---
layout: post
title:  "Production Ready Dockerfiles"
date:   2020-06-07 10:19:45 +0100
categories: ["docker"]
---

I recently refactored a Dockerfile to ensure it was ready to run a [Play][play]
application in AWS. The file was created by developers who were new to running
Docker in production, and understandably they had made some common mistakes
when working locally to make local development convenient. This post highlights
some of the issues, describing why they are problematic and how they can be
rectified.

## The Dockerfile

The Dockerfile below is an example similar to the starting point of the
refactor, with some minor tweaks.

```docker
# This Dockerfile has two required ARGs to determine which base image
# to use for the JDK and which sbt version to install.

ARG OPENJDK_TAG=8u232
FROM openjdk:${OPENJDK_TAG}

ARG SBT_VERSION=1.3.7

# Required to download keystore
ARG AWS_ACCESS_KEY_ID
ARG AWS_SECRET_ACCESS_KEY
ARG AIVEN_CERT_BUCKET

# Install sbt
RUN \
  curl -L -o sbt-$SBT_VERSION.deb https://dl.bintray.com/sbt/debian/sbt-$SBT_VERSION.deb && \
  dpkg -i sbt-$SBT_VERSION.deb && \
  rm sbt-$SBT_VERSION.deb && \
  apt-get update && \
  apt-get install sbt && \
  sbt sbtVersion && \
  useradd -d /home/app app && \
  mkdir -p /home/app && chown app:app /home/app

# Install aws-cli
RUN apt-get install -y python-pip && \
    pip install awscli


USER app
COPY --chown=app . /home/app/
WORKDIR /home/app/
ENV AIVEN_KAFKA_SECURITY_PROTOCOL="SSL"
ENV AIVEN_KAFKA_SSL_KEYSTORE_REMOTE_LOCATION="s3://$AIVEN_CERT_BUCKET/keystore.p12"
ENV AIVEN_KAFKA_SSL_KEYSTORE_LOCATION="conf/aiven-certificates/keystore.p12"
ENV AIVEN_KAFKA_SSL_TRUSTSTORE_REMOTE_LOCATION="s3://$AIVEN_CERT_BUCKET/truststore.jks"
ENV AIVEN_KAFKA_SSL_TRUSTSTORE_LOCATION="conf/aiven-certificates/truststore.jks"
RUN ./scripts/download-keystores-and-truststore.sh && sbt compile

EXPOSE 8080
CMD ["sbt", "compile", "runProd 8080"]
`

## 

[play]: https://www.playframework.com/
