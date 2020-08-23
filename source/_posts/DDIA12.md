---
title: Note for DDIA in Chapter 12
date: 2020-08-21 12:11:00
tags: note
---

## The Future of Data Systems

### Data Integration

There is unlikely to be one piece of software that is suitable for all the different circumstances in which the data is used, so you inevitably end up having to cobble together several different pieces of software in order to provide your application’s functionality.

#### Combining Specialized Tools by Deriving Data

The need for data integration often only becomes apparent if you zoom out and consider the dataflows across an entire organization.

##### Reasoning about dataflows

If it is possible for you to funnel all user input through a single system that decides on an ordering for all writes, it becomes much easier to derive other representations of the data by processing the writes in the same order. This is an application of the state machine replication approach that we saw in “Total Order Broadcast” on page 348. Whether you use change data capture or an event sourcing log is less important than simply the principle of deciding on a total order.

##### Derived data versus distributed transactions

Distributed transactions decide on an ordering of writes by using locks for mutual exclusion, while CDC and event sourcing use a log for ordering. Distributed transactions use atomic commit to ensure that changes take effect exactly once, while log-based systems are often based on deterministic retry and idempotence.

The biggest difference is that transaction systems usually provide linearizability, which implies useful guarantees such as reading your own writes. On the other hand, derived data systems are often updated asynchronously, and so they do not by default offer the same timing guarantees.

In the absence of widespread support for a good distributed transaction protocol, I believe that log-based derived data is the most promising approach for integrating different data systems. 

##### The limits of total ordering

As systems are scaled toward bigger and more complex workloads, limitations begin to emerge:

- In most cases, constructing a totally ordered log requires all events to pass through a single leader node that decides on the ordering. If the throughput of events is greater than a single machine can handle, you need to partition it across multiple machines. The order of events in two different partitions is then ambiguous.

- If the servers are spread across multiple *geographically distributed* datacenters. This implies an undefined ordering of events that originate in two different datacenters.

- When applications are deployed as *microservices*, a common design choice is to deploy each service and its durable state as an independent unit, with no durable state shared between services. When two events originate in different services, there is no defined order for those events.

- Some applications maintain client-side state that is updated immediately on user input. With such applications, clients and servers are very likely to see events in different orders.

In formal terms, deciding on a total order of events is known as *total order broadcast*, which is equivalent to consensus. Most consensus algorithms are designed for situations in which the throughput of a single node is sufficient to process the entire stream of events, and these algorithms do not provide a mechanism for multiple nodes to share the work of ordering the events. 

##### Ordering events to capture causality
...
#### Batch and Stream Processing

Batch and stream processing have a lot of principles in common, and the main fundamental difference is that stream process‐ ors operate on unbounded datasets whereas batch process inputs are of a known, finite size. There are also many detailed differences in the ways the processing engines are implemented, but these distinctions are beginning to blur.

##### Maintaining derived state

##### Reprocessing data for application evolution

In particular, reprocessing existing data provides a good mechanism for maintaining a system, evolving it to support new features and changed requirements. On the other hand, with reprocessing it is possible to restructure a dataset into a completely different model in order to better serve new requirements.

Derived views allow *gradual* evolution. 

The beauty of such a gradual migration is that every stage of the process is easily reversible if something goes wrong: you always have a working system to go back to. By reducing the risk of irreversible damage, you can be more confident about going ahead, and thus move faster to improve your system.

##### The lambda architecture

The lambda architecture proposes running two different systems in parallel: a batch processing system such as Hadoop MapReduce, and a separate stream- processing system such as Storm.

In the lambda approach, the stream processor consumes the events and quickly produces an approximate update to the view; the batch processor later consumes the *same* set of events and produces a corrected version of the derived view. 

It has a number of practical problems:

- Having to maintain the same logic to run both in a batch and in a stream pro‐ cessing framework is significant additional effort. 

- Since the stream pipeline and the batch pipeline produce separate outputs, they need to be merged in order to respond to user requests.

- Although it is great to have the ability to reprocess the entire historical dataset, doing so frequently is expensive on large datasets. 

##### Unifying batch and stream processing

More recent work has enabled the benefits of the lambda architecture to be enjoyed without its downsides, by allowing both batch computations (reprocessing historical data) and stream computations (processing events as they arrive) to be implemented in the same system.

Unifying batch and stream processing in one system requires the following features, which are becoming increasingly widely available:

- The ability to replay historical events through the same processing engine that handles the stream of recent events.

- Exactly-once semantics for stream processors—that is, ensuring that the output is the same as if no faults had occurred, even if faults did in fact occur. Like with batch processing, this requires discarding the partial output of any failed tasks.

- Tools for windowing by event time, not by processing time, since processing time is meaningless when reprocessing historical events.

### Unbundling Databases

#### Composing Data Storage Technologies

##### Creating an index

##### The meta-database of everything

In this light, I think that the dataflow across an entire organization starts looking like one huge database. 

Where will these developments take us in the future? If we start from the premise that there is no single data model or storage format that is suitable for all access pat‐ terns, I speculate that there are two avenues by which different storage and process‐ ing tools can nevertheless be composed into a cohesive system:

*Federated databases: unifying reads*
- It is possible to provide a unified query interface to a wide variety of underlying storage engines and processing methods—an approach known as a *federated database* or *polystore*.

*Unbundled databases: unifying writes*
- While federation addresses read-only querying across several different systems, it does not have a good answer to synchronizing writes across those systems. Making it easier to reliably plug together storage systems (e.g., through change data capture and event logs) is like *unbundling* a database’s index-maintenance features in a way that can synchronize writes across disparate technologies

##### Making unbundling work

Federation and unbundling are two sides of the same coin: composing a reliable, scalable, and maintainable system out of diverse components.

Transactions within a single storage or stream processing system are feasible, but when data crosses the boundary between different technologies, I believe that an asynchronous event log with idempotent writes is a much more robust and practical approach.

An ordered log of events with idempotent consumers (see “Idempotence” on page 478) is a much simpler abstrac‐ tion, and thus much more feasible to implement across heterogeneous systems.

The big advantage of log-based integration is *loose coupling* between the various components, which manifests itself in two ways:

1. At a system level, asynchronous event streams make the system as a whole more robust to outages or performance degradation of individual components. If a consumer runs slow or fails, the event log can buffer messages, allowing the producer and any other consumers to continue running unaffected.

2. At a human level, unbundling data systems allows different software components and services to be developed, improved, and maintained independently from each other by different teams. Specialization allows each team to focus on doing one thing well, with well-defined interfaces to other teams’ systems. Event logs provide an interface that is powerful enough to capture fairly strong consistency properties (due to durability and ordering of events), but also general enough to be applicable to almost any kind of data.

##### Unbundled versus integrated systems
...
##### What’s missing?

The tools for composing data systems are getting better, but I think one major part is missing: we don’t yet have the unbundled-database equivalent of the Unix shell (i.e., a high-level language for composing storage and processing systems in a simple and declarative way).

Similarly, it would be great to be able to precompute and update caches more easily.

#### Designing Applications Around Dataflow

##### Application code as a derivation function

##### Separation of application code and state

I think it makes sense to have some parts of a system that specialize in durable data storage, and other parts that specialize in running application code. The two can interact while still remaining independent.

Most web applications today are deployed as stateless services, in which any user request can be routed to any application server, and the server forgets everything about the request once it has sent the response. 

In this typical web application model, the database acts as a kind of mutable shared variable that can be accessed synchronously over the network. 

However, in most programming languages you cannot subscribe to changes in a mutable variable—you can only read it periodically. 

##### Dataflow: Interplay between state changes and application code

Thinking about applications in terms of dataflow implies renegotiating the relation‐ ship between application code and state management. Instead of treating a database as a passive variable that is manipulated by the application, we think much more about the interplay and collaboration between state, state changes, and code that processes them. Application code responds to state changes in one place by triggering state changes in another place.

Unbundling the database means taking this idea and applying it to the creation of derived datasets outside of the primary database: caches, full-text search indexes, machine learning, or analytics systems. We can use stream processing and messaging systems for this purpose.

- When maintaining derived data, the order of state changes is often important (if several views are derived from an event log, they need to process the events in the same order so that they remain consistent with each other).

- Fault tolerance is key for derived data: losing just a single message causes the derived dataset to go permanently out of sync with its data source. Both message delivery and derived state updates must be reliable.

Stable message ordering and fault-tolerant message processing are quite stringent demands, but they are much less expensive and more operationally robust than dis‐ tributed transactions. 

##### Stream processors and services

Composing stream operators into dataflow systems has a lot of similar characteristics to the microservices approach. However, the underlying communication mecha‐ nism is very different: one-directional, asynchronous message streams rather than synchronous request/response interactions.

- In the microservices approach, the code that processes the purchase would prob‐ ably query an exchange-rate service or database in order to obtain the current rate for a particular currency.

- In the dataflow approach, the code that processes purchases would subscribe to a stream of exchange rate updates ahead of time, and record the current rate in a local database whenever it changes. When it comes to processing the purchase, it only needs to query the local database.

Subscribing to a stream of changes, rather than querying the current state when needed, brings us closer to a spreadsheet-like model of computation: when some piece of data changes, any derived data that depends on it can swiftly be updated.

#### Observing Derived State


...
