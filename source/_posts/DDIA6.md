---
title: Note for DDIA in Chapter 6
date: 2020-08-13 21:43:00
tags: note
---

## Partitioning

In this chapter we will first look at different approaches for partitioning large datasets and observe how the indexing of data interacts with partitioning. We’ll then talk about rebalancing, which is necessary if you want to add or remove nodes in your cluster. Finally, we’ll get an overview of how databases route requests to the right par‐ titions and execute queries.

### Partitioning and Replication

### Partitioning of Key-Value Data

#### Partitioning by Key Range

This partitioning strategy is used by Bigtable, its open source equivalent HBase, RethinkDB, and MongoDB before version 2.4.

Within each partition, we can keep keys in sorted order. This has the advantage that range scans are easy, and you can treat the key as a concatenated index in order to fetch several related records in one query.

However, the downside of key range partitioning is that certain access patterns can lead to *hot spots*.

#### Partitioning by Hash of Key

A good hash function takes skewed data and makes it uniformly distributed. 

For partitioning purposes, the hash function need not be cryptographically strong: for example, Cassandra and MongoDB use MD5, and Voldemort uses the Fowler– Noll–Vo function. 

Unfortunately however, by using the hash of the key for partitioning we lose a nice property of key-range partitioning: the ability to do efficient range queries. 

#### Skewed Workloads and Relieving Hot Spots

### Partitioning and Secondary Indexes

The problem with secondary indexes is that they don’t map neatly to partitions. There are two main approaches to partitioning a database with secondary indexes: document-based partitioning and term-based partitioning.

#### Partitioning Secondary Indexes by Document

In this indexing approach, each partition is completely separate: each partition main‐ tains its own secondary indexes, covering only the documents in that partition. For that reason, a document-partitioned index is also known as a local index (as opposed to a global index, described in the next section).

You need to send the query to all partitions, and combine all the results you get back.

This approach to querying a partitioned database is sometimes known as *scatter/ gather*, and it can make read queries on secondary indexes quite expensive. 

#### Partitioning Secondary Indexes by Term

A global index must also be partitioned, but it can be partitioned differently from the primary key index.

We call this kind of index *term-partitioned*, because the term we’re looking for determines the partition of the index. 

The advantage of a global (term-partitioned) index over a document-partitioned index is that it can make reads more efficient: rather than doing scatter/gather over all partitions, a client only needs to make a request to the partition containing the term that it wants. However, the downside of a global index is that writes are slower and more complicated, because a write to a single document may now affect multiple partitions of the index (every term in the document might be on a different partition, on a different node).

In practice, updates to global secondary indexes are often asynchronous (that is, if you read the index shortly after a write, the change you just made may not yet be reflected in the index).

### Rebalancing Partitions

The process of moving load from one node in the cluster to another is called *rebalancing*.

No matter which partitioning scheme is used, rebalancing is usually expected to meet some minimum requirements:

- After rebalancing, the load (data storage, read and write requests) should be shared fairly between the nodes in the cluster.

- While rebalancing is happening, the database should continue accepting reads and writes.

- No more data than necessary should be moved between nodes, to make rebalancing fast and to minimize the network and disk I/O load.

#### Strategies for Rebalancing

##### How not to do it: hash mod N

When partitioning by the hash of a key, we said earlier that it’s best to divide the possible hashes into ranges and assign each range to a partition (e.g., assign key to partition 0 if 0 ≤ hash(key) < b0, to partition 1 if b0 ≤ hash(key) < b1, etc.).

The problem with the mod N approach is that if the number of nodes N changes, most of the keys will need to be moved from one node to another. 

##### Fixed number of partitions

Fortunately, there is a fairly simple solution: create many more partitions than there are nodes, and assign several partitions to each node.  For example, a database running on a cluster of 10 nodes may be split into 1,000 partitions from the outset so that approximately 100 partitions are assigned to each node.

Now, if a node is added to the cluster, the new node can *steal* a few partitions from every existing node until partitions are fairly distributed once again. If a node is removed from the cluster, the same happens in reverse.

In this configuration, the number of partitions is usually fixed when the database is first set up and not changed afterward. 

##### Dynamic partitioning

For databases that use key range partitioning, a fixed number of partitions with fixed boundaries would be very incon‐ venient: if you got the boundaries wrong, you could end up with all of the data in one partition and all of the other partitions empty.

When a partition grows to exceed a configured size (on HBase, the default is 10 GB), it is split into two partitions so that approximately half of the data ends up on each side of the split. Conversely, if lots of data is deleted and a partition shrinks below some threshold, it can be merged with an adjacent par‐ tition. This process is similar to what happens at the top level of a B-tree.

Each partition is assigned to one node, and each node can handle multiple partitions, like in the case of a fixed number of partitions. After a large partition has been split, one of its two halves can be transferred to another node in order to balance the load.

An advantage of dynamic partitioning is that the number of partitions adapts to the total data volume. 

However, a caveat is that an empty database starts off with a single partition, since there is no a priori information about where to draw the partition boundaries. While the dataset is small—until it hits the point at which the first partition is split—all writes have to be processed by a single node while the other nodes sit idle. 

Dynamic partitioning is not only suitable for key range–partitioned data, but can equally well be used with hash-partitioned data. 

##### Partitioning proportionally to nodes

With dynamic partitioning, the number of partitions is proportional to the size of the dataset, since the splitting and merging processes keep the size of each partition between some fixed minimum and maximum. On the other hand, with a fixed number of partitions, the size of each partition is proportional to the size of the dataset. In both of these cases, the number of partitions is independent of the number of nodes.

A third option, used by Cassandra and Ketama, is to make the number of partitions proportional to the number of nodes—in other words, to have a fixed number of partitions *per node*.

When a new node joins the cluster, it randomly chooses a fixed number of existing partitions to split, and then takes ownership of one half of each of those split partitions while leaving the other half of each partition in place. 

Picking partition boundaries randomly requires that hash-based partitioning is used. ndeed, this approach corresponds most closely to the original definition of consistent hashing. Newer hash func‐ tions can achieve a similar effect with lower metadata overhead.

#### Operations: Automatic or Manual Rebalancing

### Request Routing

Somebody needs to stay on top of those changes in order to answer the question: if I want to read or write the key “foo”, which IP address and port number do I need to connect to?

This is an instance of a more general problem called *service discovery*, which isn’t limited to just databases. 

On a high level, there are a few different approaches to this problem:

1. Allow clients to contact any node (e.g., via a round-robin load balancer). If that node coincidentally owns the partition to which the request applies, it can handle the request directly; otherwise, it forwards the request to the appropriate node, receives the reply, and passes the reply along to the client.

2. Send all requests from clients to a routing tier first, which determines the node that should handle each request and forwards it accordingly. This routing tier does not itself handle any requests; it only acts as a partition-aware load balancer.

3. Require that clients be aware of the partitioning and the assignment of partitions to nodes. In this case, a client can connect directly to the appropriate node, without any intermediary.

In all cases, the key problem is: how does the component making the routing decision (which may be one of the nodes, or the routing tier, or the client) learn about changes in the assignment of partitions to nodes?

This is a challenging problem, because it is important that all participants agree— otherwise requests would be sent to the wrong nodes and not handled correctly.

Many distributed data systems rely on a separate coordination service such as ZooKeeper to keep track of this cluster metadata. Each node registers itself in ZooKeeper, and ZooKeeper maintains the authoritative mapping of partitions to nodes. Other actors, such as the routing tier or the partitioning-aware client, can subscribe to this information in ZooKeeper. Whenever a partition changes ownership, or a node is added or removed, ZooKeeper notifies the routing tier so that it can keep its routing information up to date.

Cassandra and Riak take a different approach: they use a gossip protocol among the nodes to disseminate any changes in cluster state. Requests can be sent to any node, and that node forwards them to the appropriate node for the requested partition.

#### Parallel Query Execution

So far we have focused on very simple queries that read or write a single key.

However, *massively parallel processing* (MPP) relational database products, often used for analytics, are much more sophisticated in the types of queries they support. The MPP query optimizer breaks this complex query into a num‐ ber of execution stages and partitions, many of which can be executed in parallel on different nodes of the database cluster. 

### Summary

...
