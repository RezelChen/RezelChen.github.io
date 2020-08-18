---
title: Note for DDIA in Chapter 9
date: 2020-08-17 23:39:00
tags: note
---

## Consistency and Consensus

The best way of building fault-tolerant systems is to find some general-purpose abstractions with useful guarantees, implement them once, and then let applications rely on those guarantees.

One of the most important abstractions for distributed systems is consensus: that is, getting all of the nodes to agree on something.

The limits of what is and isn’t possible have been explored in depth, both in theoretical proofs and in practical implementations. We will get an overview of those fundamental limits in this chapter.

### Consistency Guarantees

A better name for *eventual consistency* may be *convergence*, as we expect all replicas to eventually converge to the same value.

There is some similarity between distributed consistency models and the hierarchy of transaction isolation levels we discussed previously. But while there is some overlap, they are mostly independent con‐ cerns: transaction isolation is primarily about avoiding race conditions due to concurrently executing transactions, whereas distributed consistency is mostly about coordinating the state of replicas in the face of delays and faults.

This chapter covers a broad range of topics, but as we shall see, these areas are in fact deeply linked:

- We will start by looking at one of the strongest consistency models in common use, *linearizability*, and examine its pros and cons.

- We’ll then examine the issue of ordering events in a distributed system, particularly around causality and total ordering.

- In the third section we will explore how to atomically commit a distributed transaction, which will finally lead us toward solutions for the consensus problem.

### Linearizability

The exact definition of linearizability is quite subtle, and we will explore it in the rest of this section. But the basic idea is to make a system appear as if there were only one copy of the data, and all operations on it are atomic. With this guarantee, even though there may be multiple replicas in reality, the application does not need to worry about them.

#### What Makes a System Linearizable?

The requirement of linearizability is that the lines joining up the operation markers always move forward in time (from left to right), never backward. This requirement ensures the recency guarantee we discussed earlier: once a new value has been written or read, all subsequent reads see the value that was written, until it is overwritten again.

Linearizability Versus Serializability: 

- Serializability is an isolation property of transactions, where every transaction may read and write multiple objects. It guarantees that transactions behave the same as if they had executed in some serial order (each transaction running to completion before the next transaction starts). It is okay for that serial order to be different from the order in which transactions were actually run.

- Linearizability is a recency guarantee on reads and writes of a register (an individual object). It doesn’t group operations together into transactions, so it does not prevent problems such as write skew, unless you take additional measures such as materializing conflicts.

Implementations of serializability based on two-phase locking or actual serial execution are typically linearizable.

However, serializable snapshot isolation is not linearizable: by design, it makes reads from a consistent snapshot, to avoid lock contention between readers and writers. The whole point of a consistent snapshot is that it does not include writes that are more recent than the snapshot, and thus reads from the snapshot are not linearizable.

#### Relying on Linearizability

##### Locking and leader election

A system that uses single-leader replication needs to ensure that there is indeed only one leader, not several (split brain). One way of electing a leader is to use a lock: every node that starts up tries to acquire the lock, and the one that succeeds becomes the leader.

Distributed locking is also used at a much more granular level in some distributed databases, such as Oracle Real Application Clusters (RAC). RAC uses a lock per disk page, with multiple nodes sharing access to the same disk storage system.

##### Constraints and uniqueness guarantees

This situation is actually similar to a lock: when a user registers for your service, you can think of them acquiring a “lock” on their chosen username. The operation is also very similar to an atomic compare-and-set, setting the username to the ID of the user who claimed it, provided that the username is not already taken.

##### Cross-channel timing dependencies

#### Implementing Linearizable Systems

The most common approach to making a system fault-tolerant is to use replication. Let’s revisit the replication methods from Chapter 5, and compare whether they can be made linearizable:

- Single-leader replication (potentially linearizable)
- Consensus algorithms (linearizable)
- Multi-leader replication (not linearizable)
- Leaderless replication (probably not linearizable)

##### Linearizability and quorums

It is possible to make Dynamo-style quorums linearizable at the cost of reduced performance: a reader must perform *read repair* synchronously, before returning results to the application, and a writer must read the latest state of a quorum of nodes before sending its writes.

Moreover, only linearizable read and write operations can be implemented in this way; a linearizable compare-and-set operation cannot, because it requires a consensus algorithm.

In summary, it is safest to assume that a leaderless system with Dynamo-style replication does not provide linearizability.

#### The Cost of Linearizability

##### The CAP theorem

The trade-off is as follows:

- If your application *requires* linearizability, and some replicas are disconnected from the other replicas due to a network problem, then some replicas cannot process requests while they are disconnected: they must either wait until the network problem is fixed, or return an error (either way, they become *unavailable*).

- If your application *does not require* linearizability, then it can be written in a way that each replica can process requests independently, even if it is disconnected from other replicas (e.g., multi-leader). In this case, the application can remain *available* in the face of a network problem, but its behavior is not linearizable.

CAP is sometimes presented as *Consistency, Availability, Partition tolerance: pick 2 out of 3*. Unfortunately, putting it this way is misleading because network partitions are a kind of fault, so they aren’t something about which you have a choice: they will happen whether you like it or not.

Thus, a better way of phrasing CAP would be *either Consistent or Available when Partitioned*.

##### Linearizability and network delays

Although linearizability is a useful guarantee, surprisingly few systems are actually linearizable in practice.

The reason for dropping linearizability is *performance*, not fault tolerance.

In Chapter 12 we will discuss some approaches for avoiding linearizability without sacrificing correctness.

### Ordering Guarantees

It turns out that there are deep connections between ordering, linearizability, and consensus. 

#### Ordering and Causality

There are several reasons why ordering keeps coming up, and one of the reasons is that it helps preserve *causality*. 

Causality imposes an ordering on events: cause comes before effect; a message is sent before that message is received; the question comes before the answer. 

If a system obeys the ordering imposed by causality, we say that it is *causally consistent*. For example, snapshot isolation provides causal consistency: when you read from the database, and you see some piece of data, then you must also be able to see any data that causally precedes it.

##### The causal order is not a total order

The difference between a *total order* and a *partial order* is reflected in different data‐ base consistency models:

- Linearizability: In a linearizable system, we have a total order of operations: if the system behaves as if there is only a single copy of the data, and every operation is atomic, this means that for any two operations we can always say which one happened first.

- Causality: We said that two operations are concurrent if neither happened before the other. Put another way, two events are ordered if they are causally related (one happened before the other), but they are incomparable if they are concurrent. This means that causality defines a *partial order*, not a total order: some operations are ordered with respect to each other, but some are incomparable.

If you are familiar with distributed version control systems such as Git, their version histories are very much like the graph of causal dependencies. Often one commit happens after another, in a straight line, but sometimes you get branches (when sev‐ eral people concurrently work on a project), and merges are created when those con‐ currently created commits are combined.

##### Linearizability is stronger than causal consistency

Any system that is linearizable will preserve cau‐ sality correctly.

The good news is that a middle ground is possible. Linearizability is not the only way of preserving causality—there are other ways too. In fact, causal consistency is the strongest possible consistency model that does not slow down due to network delays, and remains available in the face of network failures.

In many cases, systems that appear to require linearizability in fact only really require causal consistency, which can be implemented more efficiently. Based on this obser‐ vation, researchers are exploring new kinds of databases that preserve causality, with performance and availability characteristics that are similar to those of eventually consistent systems.

##### Capturing causal dependencies

The techniques for determining which operation happened before which other oper‐ ation are similar to what we discussed in “Detecting Concurrent Writes” on page 184.

In order to determine the causal ordering, the database needs to know which version of the data was read by the application. This is why, in Figure 5-13, the version num‐ ber from the prior operation is passed back to the database on a write. A similar idea appears in the conflict detection of SSI, as discussed in “Serializable Snapshot Isolation (SSI)” on page 261: when a transaction wants to commit, the database checks whether the version of the data that it read is still up to date. To this end, the database keeps track of which data has been read by which transaction.

#### Sequence Number Ordering

Although causality is an important theoretical concept, actually keeping track of all causal dependencies can become impractical.

However, there is a better way: we can use *sequence numbers* or *timestamps* to order events. A timestamp need not come from a time-of-day clock (or physical clock, which have many problems, as discussed in “Unreliable Clocks” on page 287). It can instead come from a logical clock, which is an algorithm to generate a sequence of numbers to identify operations, typically using counters that are incremented for every operation.

In particular, we can create sequence numbers in a total order that is *consistent with causality*: we promise that if operation A causally happened before B, then A occurs before B in the total order.

In a database with single-leader replication (see “Leaders and Followers” on page 152), the replication log defines a total order of write operations that is consistent with causality. The leader can simply increment a counter for each operation, and thus assign a monotonically increasing sequence number to each operation in the replication log. If a follower applies the writes in the order they appear in the replication log, the state of the follower is always causally consistent (even if it is lagging behind the leader).

##### Noncausal sequence number generators

##### Lamport timestamps

A Lamport timestamp bears no relationship to a physical time-of-day clock, but it provides total ordering: if you have two timestamps, the one with a greater counter value is the greater timestamp; if the counter values are the same, the one with the greater node ID is the greater timestamp.

The key idea about Lamport timestamps, which makes them consis‐ tent with causality, is the following: every node and every client keeps track of the *maximum* counter value it has seen so far, and includes that maximum on every request. When a node receives a request or response with a maximum counter value greater than its own counter value, it immediately increases its own counter to that maximum.

As long as the maximum counter value is carried along with every operation, this scheme ensures that the ordering from the Lamport timestamps is consistent with causality, because every causal dependency results in an increased timestamp.

Lamport timestamps are sometimes confused with version vectors, which we saw in “Detecting Concurrent Writes” on page 184. Although there are some similarities, they have a different purpose: version vectors can distinguish whether two operations are concurrent or whether one is causally dependent on the other, whereas Lamport timestamps always enforce a total ordering. 

From the total ordering of Lamport timestamps, you cannot tell whether two operations are concurrent or whether they are causally dependent. The advantage of Lamport timestamps over version vectors is that they are more compact.

##### Timestamp ordering is not sufficient

The problem here is that the total order of operations only emerges after you have collected all of the operations. 

If you have an operation to create a username, and you are sure that no other node can insert a claim for the same username ahead of your operation in the total order, then you can safely declare the operation successful. This idea of knowing when your total order is finalized is captured in the topic of *total order broadcast*.

#### Total Order Broadcast

Total order broadcast is usually described as a protocol for exchanging messages between nodes. Informally, it requires that two safety properties always be satisfied:

- Reliable delivery: No messages are lost: if a message is delivered to one node, it is delivered to all nodes.

- Totally ordered delivery: Messages are delivered to every node in the same order.

##### Using total order broadcast

Consensus services such as ZooKeeper and etcd actually implement total order broadcast. This fact is a hint that there is a strong connection between total order broadcast and consensus, which we will explore later in this chapter.

An important aspect of total order broadcast is that the order is fixed at the time the messages are delivered.

Another way of looking at total order broadcast is that it is a way of creating a log (as in a replication log, transaction log, or write-ahead log): delivering a message is like appending to the log. Since all nodes must deliver the same messages in the same order, all nodes can read the log and see the same sequence of messages.

Total order broadcast is also useful for implementing a lock service that provides *fencing tokens*. Every request to acquire the lock is appended as a message to the log, and all messages are sequentially numbered in the order they appear in the log. The sequence number can then serve as a fencing token, because it is monotonically increasing. 

##### Implementing linearizable storage using total order broadcast

Total order broadcast is asynchronous: messages are guaranteed to be delivered relia‐ bly in a fixed order, but there is no guarantee about when a message will be delivered (so one recipient may lag behind the others). By contrast, linearizability is a recency guarantee: a read is guaranteed to see the latest value written.

You can implement such a linearizable compare-and-set operation as follows by using total order broadcast as an append-only log:

1. Append a message to the log, tentatively indicating the username you want to claim.

2. Read the log, and wait for the message you appended to be delivered back to you.

3. Check for any messages claiming the username that you want. If the first message for your desired username is your own message, then you are successful; otherwise, you abort the operation.

##### Implementing total order broadcast using linearizable storage

The algorithm is simple: for every message you want to send through total order broadcast, you increment-and-get the linearizable integer, and then attach the value you got from the register as a sequence number to the message. You can then send the message to all nodes (resending any lost messages), and the recipients will deliver the messages consecutively by sequence number.

Note that unlike Lamport timestamps, the numbers you get from incrementing the linearizable register form a sequence with no gaps. Thus, if a node has delivered mes‐ sage 4 and receives an incoming message with a sequence number of 6, it knows that it must wait for message 5 before it can deliver message 6. The same is not the case with Lamport timestamps -— in fact, this is the key difference between total order broadcast and timestamp ordering.

It can be proved that a linearizable compare-and-set (or increment-and-get) register and total order broadcast are both equivalent to consensus.

### Distributed Transactions and Consensus

#### Atomic Commit and Two-Phase Commit (2PC)

##### From single-node to distributed atomic commit

A node must only commit once it is certain that all other nodes in the transaction are also going to commit.

##### Introduction to two-phase commit

2PC uses a new component that does not normally appear in single-node transactions: a coordinator (also known as transaction manager). The coordinator is often implemented as a library within the same application process that is requesting the transaction, but it can also be a separate pro‐ cess or service.

A 2PC transaction begins with the application reading and writing data on multiple database nodes, as normal. We call these database nodes participants in the transac‐ tion. When the application is ready to commit, the coordinator begins phase 1: it sends a prepare request to each of the nodes, asking them whether they are able to commit. The coordinator then tracks the responses from the participants:

- If all participants reply “yes,” indicating they are ready to commit, then the coor‐ dinator sends out a commit request in phase 2, and the commit actually takes place.

- If any of the participants replies “no,” the coordinator sends an abort request to all nodes in phase 2.

##### A system of promises

Thus, the protocol contains two crucial “points of no return”: when a participant votes “yes,” it promises that it will definitely be able to commit later (although the coordinator may still choose to abort); and once the coordinator decides, that deci‐ sion is irrevocable. Those promises ensure the atomicity of 2PC.

##### Coordinator failure

We have discussed what happens if one of the participants or the network fails during 2PC: if any of the prepare requests fail or time out, the coordinator aborts the trans‐ action; if any of the commit or abort requests fail, the coordinator retries them indefi‐ nitely. 

##### Three-phase commit

Two-phase commit is called a *blocking* atomic commit protocol due to the fact that 2PC can become stuck waiting for the coordinator to recover. In theory, it is possible to make an atomic commit protocol *nonblocking*, so that it does not get stuck if a node fails. However, making this work in practice is not so straightforward.

#### Distributed Transactions in Practice

##### Exactly-once message processing

##### XA transactions

*X/Open XA* (short for eXtended Architecture) is a standard for implementing two- phase commit across heterogeneous technologies.

##### Holding locks while in doubt

##### Recovering from coordinator failure

Many XA implementations have an emergency escape hatch called *heuristic decisions*: allowing a participant to unilaterally decide to abort or commit an in-doubt transaction without a definitive decision from the coordinator. To be clear, *heuristic* here is a euphemism for *probably breaking atomicity*, since it violates the system of promises in two-phase commit. 

##### Limitations of distributed transactions

XA transactions solve the real and important problem of keeping several participant data systems consistent with each other, but as we have seen, they also introduce major operational problems. In particular, the key realization is that the transaction coordinator is itself a kind of database (in which transaction outcomes are stored), and so it needs to be approached with the same care as any other important database:

- If the coordinator is not replicated but runs only on a single machine, it is a sin‐ gle point of failure for the entire system.

- Many server-side applications are developed in a stateless model, with all persistent state stored in a database, which has the advantage that application servers can be added and removed at will. However, when the coordinator is part of the application server, it changes the nature of the deployment. Suddenly, the coordinator’s logs become a crucial part of the durable system state—as important as the databases themselves, since the coordinator logs are required in order to recover in-doubt transactions after a crash. Such application servers are no longer stateless.

- Since XA needs to be compatible with a wide range of data systems, it is necessarily a lowest common denominator.

- For database-internal distributed transactions (not XA), the limitations are not so great—for example, a distributed version of SSI is possible. However, there remains the problem that for 2PC to successfully commit a transaction, *all* participants must respond.  Consequently, if *any* part of the system is broken, the transaction also fails. Distributed transactions thus have a tendency of amplifying failures, which runs counter to our goal of building fault-tolerant systems.

#### Fault-Tolerant Consensus

The consensus problem is normally formalized as follows: one or more nodes may *propose* values, and the consensus algorithm *decides* on one of those values.

In this formalism, a consensus algorithm must satisfy the following properties:

- Uniform agreement: No two nodes decide differently.

- Integrity: No node decides twice.

- Validity: If a node decides value v, then v was proposed by some node.

- Termination: Every node that does not crash eventually decides some value.

##### Consensus algorithms and total order broadcast

The best-known fault-tolerant consensus algorithms are Viewstamped Replication (VSR), Paxos, Raft, and Zab. 

Remember that total order broadcast requires messages to be delivered exactly once, in the same order, to all nodes. If you think about it, this is equivalent to performing several rounds of consensus: in each round, nodes propose the message that they want to send next, and then decide on the next message to be delivered in the total order.

So, total order broadcast is equivalent to repeated rounds of consensus (each consen‐ sus decision corresponding to one message delivery):

- Due to the agreement property of consensus, all nodes decide to deliver the same messages in the same order.

- Due to the integrity property, messages are not duplicated.

- Due to the validity property, messages are not corrupted and not fabricated out of thin air.

- Due to the termination property, messages are not lost.

Viewstamped Replication, Raft, and Zab implement total order broadcast directly, because that is more efficient than doing repeated rounds of one-value-at-a-time consensus. In the case of Paxos, this optimization is known as Multi-Paxos.

##### Single-leader replication and consensus

It seems that in order to elect a leader, we first need a leader. In order to solve consensus, we must first solve consensus. How do we break out of this conundrum?

##### Epoch numbering and quorums

Thus, we have two rounds of voting: once to choose a leader, and a second time to vote on a leader’s proposal. The key insight is that the quorums for those two votes must overlap: if a vote on a proposal succeeds, at least one of the nodes that voted for it must have also participated in the most recent leader election.

This voting process looks superficially similar to two-phase commit. The biggest dif‐ ferences are that in 2PC the coordinator is not elected, and that fault-tolerant consen‐ sus algorithms only require votes from a majority of nodes, whereas 2PC requires a “yes” vote from *every* participant. Moreover, consensus algorithms define a recovery process by which nodes can get into a consistent state after a new leader is elected, ensuring that the safety properties are always met. These differences are key to the correctness and fault tolerance of a consensus algorithm.

##### Limitations of consensus

The process by which nodes vote on proposals before they are decided is a kind of synchronous replication. 

Consensus systems always require a strict majority to operate.

Most consensus algorithms assume a fixed set of nodes that participate in voting, which means that you can’t just add or remove nodes in the cluster. 

Consensus systems generally rely on timeouts to detect failed nodes. In environments with highly variable network delays, especially geographically distributed systems, it often happens that a node falsely believes the leader to have failed due to a transient network issue.

Sometimes, consensus algorithms are particularly sensitive to network problems.

#### Membership and Coordination Services

ZooKeeper and etcd are designed to hold small amounts of data that can fit entirely in memory (although they still write to disk for durability)—so you wouldn’t want to store all of your application’s data here. That small amount of data is replicated across all the nodes using a fault-tolerant total order broadcast algorithm. 

ZooKeeper is modeled after Google’s Chubby lock service [14, 98], implementing not only total order broadcast (and hence consensus), but also an interesting set of other features that turn out to be particularly useful when building distributed systems:

- Linearizable atomic operations

- Total ordering of operations

- Failure detection

- Change notifications

##### Allocating work to nodes

##### Service discovery

##### Membership services

### Summary
...
