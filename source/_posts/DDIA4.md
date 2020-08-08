---
title: Note for DDIA in Chapter 4
date: 2020-07-28 17:38:00
tags: note
---

## Encoding and Evolution

### Formats for Encoding Data

#### Language-Specific Formats

It’s generally a bad idea to use your language’s built-in encoding for anything other than very transient purposes.

#### JSON, XML, and Binary Variants

##### Binary encoding

#### Thrift and Protocol Buffers
...
#### Avro
...
#### The Merits of Schemas

- They can be much more compact than the various “binary JSON” variants, since they can omit field names from the encoded data.

- The schema is a valuable form of documentation, and because the schema is required for decoding, you can be sure that it is up to date (whereas manually maintained documentation may easily diverge from reality).

- Keeping a database of schemas allows you to check forward and backward com‐ patibility of schema changes, before anything is deployed.

- For users of statically typed programming languages, the ability to generate code from the schema is useful, since it enables type checking at compile time.

In summary, schema evolution allows the same kind of flexibility as schemaless/schema-on-read JSON databases provide, while also providing better guarantees about your data and better tooling.

### Modes of Dataflow

#### Dataflow Through Databases

##### Different values written at different times

Schema evolution thus allows the entire database to appear as if it was encoded with a single schema, even though the underlying storage may contain records encoded with various historical versions of the schema.

##### Archival storage

#### Dataflow Through Services: REST and RPC

##### Web services

##### The problems with remote procedure calls (RPCs)

Although RPC seems convenient at first, the approach is fundamentally flawed. A network request is very different from a local function call:

- A local function call is predictable and either succeeds or fails, depending only on parameters that are under your control. A network request is unpredictable: the request or response may be lost due to a network problem, or the remote machine may be slow or unavailable, and such problems are entirely outside of your control. 

- A network request has another possible outcome: it may return without a result, due to a timeout. 

- If you retry a failed network request, it could happen that the requests are actually getting through, and only the responses are getting lost. In that case, retrying will cause the action to be performed multiple times, unless you build a mechanism for deduplication (idempotence) into the protocol.

- A network request is much slower than a function call, and its latency is also wildly variable.

- When you call a local function, you can efficiently pass it references (pointers) to objects in local memory. When you make a network request, all those parameters need to be encoded into a sequence of bytes that can be sent over the network.

- The client and the service may be implemented in different programming lan‐ guages, so the RPC framework must translate datatypes from one language into another.

##### Current directions for RPC

The main focus of RPC frameworks is on requests between services owned by the same organization, typically within the same datacenter.

##### Data encoding and evolution for RPC

We can make a simplifying assumption in the case of dataflow through services: it is reasonable to assume that all the servers will be updated first, and all the clients second. Thus, you only need backward compatibility on requests, and forward compatibility on responses.

If a compatibility-breaking change is required, the service provider often ends up maintaining multiple versions of the service API side by side.

#### Message-Passing Dataflow

Using a message broker has several advantages compared to direct RPC:

- It can act as a buffer if the recipient is unavailable or overloaded, and thus improve system reliability.

- It can automatically redeliver messages to a process that has crashed, and thus prevent messages from being lost.

- It avoids the sender needing to know the IP address and port number of the recipient.

- It allows one message to be sent to several recipients.

- It logically decouples the sender from the recipient.

However, a difference compared to RPC is that message-passing communication is usually one-way: a sender normally doesn’t expect to receive a reply to its messages. 

##### Message brokers

##### Distributed actor frameworks

The *actor model* is a programming model for concurrency in a single process. Each actor typically represents one client or entity, it may have some local state, and it communicates with other actors by sending and receiving asynchronous messages. Message delivery is not guaranteed: in certain error scenarios, mes‐ sages will be lost. Since each actor processes only one message at a time, it doesn’t need to worry about threads, and each actor can be scheduled independently by the framework.

In *distributed actor frameworks*, this programming model is used to scale an applica‐ tion across multiple nodes. The same message-passing mechanism is used, no matter whether the sender and recipient are on the same node or different nodes. 

