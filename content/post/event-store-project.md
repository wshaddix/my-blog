---
title: "Event Store Project"
date: 2017-09-09T11:35:31-04:00
draft: true
menu: "main"
tags: [
    "microservice",
    "docker",
    "oss-project",
    ".net-core"
]
categories: [
    "Api Design",
    "Continuous Development",
    "Azure Architecture"
]
description: "An open source project to build an Event Store microservice that can be used to enable reactive workflows and performing analytics on what is happeing in any software system."
---

_NOTE: This post will be updated as more articles are written with links to the new and/or updated content_
# What is an Event Store
An event store is a microservice that allows applications to record things that happen inside their system for later analysis (stream processing or polling) and insights (capture metrics). One of the main use-cases for an event store is to enable reactive workflows between multiple (or sometimes the same) applications without having temporal coupling or api integration requirements.

# Defining the Event Store Microservice
The actors and use cases (record | record array of events in batches | query by XXX | NodaTime for extreme accuracy)
Store and forward extensibility points (azure event grid | pubnub | ???)
# Designing the Event Store Microservice
**Architecture**: Scalability in the api and the data store
Recording vs reading (separate backends, eventual consistency, etc)

**Api**: Postman collection w/ documentation & tests

# Implementing the Event Store Microservice

# Deploying the Event Store Microservice
CI/CD docker pipeline. Ability to deploy/test feature branches. Zero downtime production deployments.
Azure traffic manager (performance based on closest location)

# Monitoring & Scaling the Event Store Microservice
Throughput, performance, data size (azure api manager).
Exception management
Metrics
Logging
Alerting
Healthchecks

