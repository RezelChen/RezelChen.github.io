---
title: Note for DDIA in Chapter 1
date: 2020-05-27 22:54:40
tags: note
---

### Reliability
- The application performs the function that the user expected.
- It can tolerate the user making mistakes or using the software in unexpected ways.
- Its performance is good enough for the required use case, under the expected load and data volume.
- The system prevents any unauthorized access and abuse.

### Scalability

#### Describing Load
Load can be described with a few numbers which we call *load parameters*. The best choice of parameters depends on the architecture of your system: it may be requests per second to a web server, the ratio of reads to writes in a database, the number of simultaneously active users in a chat room, the hit rate on a cache, or something else. Perhaps the average case is what matters for you, or perhaps your bottleneck is dominated by a small number of extreme cases.

#### Describing Performance

- throughput — the number of records we can process per second
- the total time it takes to run a job on a dataset of a certain size
- response time — the time between a client sending a request and receiving a response
- latency is the duration that a request is waiting to be handled—during which it is *latent*, awaiting service

High percentiles of response times, also known as *tail latencies*, are important because they directly affect users’ experience of the service.

#### Approaches for Coping with Load

While distributing stateless services across multiple machines is fairly straightfor‐ ward, taking stateful data systems from a single node to a distributed setup can intro‐ duce a lot of additional complexity.

An architecture that scales well for a particular application is built around assumptions of which operations will be common and which will be rare — the load parameters.

Even though they are specific to a particular application, scalable architectures are nevertheless usually built from general-purpose building blocks, arranged in familiar patterns.

### Maintainability

#### Operability: Making Life Easy for Operations

#### Simplicity: Managing Complexity

#### Evolvability: Making Change Easy
