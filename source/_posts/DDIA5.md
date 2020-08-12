---
title: Note for DDIA in Chapter 5
date: 2020-08-08 20:23:00
tags: note
---

## Replication

There are several reasons why you might want to replicate data:

- To keep data geographically close to your users (and thus reduce latency)

- To allow the system to continue working even if some of its parts have failed (and thus increase availability)

- To scale out the number of machines that can serve read queries (and thus increase read throughput)

 We will discuss three popular algorithms for replicating changes between nodes: *single-leader*, *multi-leader*, and *leaderless* replication. Almost all distributed databases use one of these three approaches. 

### Leaders and Followers

####  Synchronous Versus Asynchronous Replication

In practice, if you enable synchronous replication on a database, it usually means that *one* of the followers is synchronous, and the others are asynchronous.

#### Setting Up New Followers

#### Handling Node Outages

##### Follower failure: Catch-up recovery

##### Leader failure: Failover

An automatic failover process usually consists of the following steps:

- *Determining that the leader has failed.*

- *Choosing a new leader.* The best candidate for leadership is usually the replica with the most up-to-date data changes from the old leader (to minimize any data loss).

- *Reconfiguring the system to use the new leader.*


Failover is fraught with things that can go wrong:

- If asynchronous replication is used, the new leader may not have received all the writes from the old leader before it failed. If the former leader rejoins the cluster after a new leader has been chosen, what should happen to those writes? The new leader may have received conflicting writes in the meantime. The most common solution is for the old leader’s unreplicated writes to simply be discarded, which may violate clients’ durability expectations.

- Discarding writes is especially dangerous if other storage systems outside of the database need to be coordinated with the database contents. 

- In certain fault scenarios (see Chapter 8), it could happen that two nodes both believe that they are the leader. This situation is called split brain, and it is dan‐ gerous: if both leaders accept writes, and there is no process for resolving con‐ flicts (see “Multi-Leader Replication” on page 168), data is likely to be lost or corrupted. 

- What is the right timeout before the leader is declared dead? 

#### Implementation of Replication Logs

##### Statement-based replication

In the simplest case, the leader logs every write request (statement) that it executes and sends that statement log to its followers. 

Although this may sound reasonable, there are various ways in which this approach to replication can break down:

- Any statement that calls a nondeterministic function, such as NOW() to get the current date and time or RAND() to get a random number, is likely to generate a different value on each replica.

- If statements use an autoincrementing column, or if they depend on the existing data in the database (e.g., UPDATE ... WHERE <some condition>), they must be executed in exactly the same order on each replica, or else they may have a differ‐ ent effect. This can be limiting when there are multiple concurrently executing transactions.

- Statements that have side effects (e.g., triggers, stored procedures, user-defined functions) may result in different side effects occurring on each replica, unless the side effects are absolutely deterministic.

##### Write-ahead log (WAL) shipping

The main disadvantage is that the log describes the data on a very low level: a WAL con‐ tains details of which bytes were changed in which disk blocks. This makes replica‐ tion closely coupled to the storage engine. 

That may seem like a minor implementation detail, but it can have a big operational impact. If the replication protocol allows the follower to use a newer software version than the leader, you can perform a zero-downtime upgrade of the database software by first upgrading the followers and then performing a failover to make one of the upgraded nodes the new leader. If the replication protocol does not allow this version mismatch, as is often the case with WAL shipping, such upgrades require downtime.

##### Logical (row-based) log replication

An alternative is to use different log formats for replication and for the storage engine, which allows the replication log to be decoupled from the storage engine internals. This kind of replication log is called a logical log, to distinguish it from the storage engine’s (physical) data representation.

A logical log for a relational database is usually a sequence of records describing writes to database tables at the granularity of a row:

- For an inserted row, the log contains the new values of all columns.

- For a deleted row, the log contains enough information to uniquely identify the row that was deleted. Typically this would be the primary key, but if there is no primary key on the table, the old values of all columns need to be logged.

- For an updated row, the log contains enough information to uniquely identify the updated row, and the new values of all columns (or at least the new values of all columns that changed).

A transaction that modifies several rows generates several such log records, followed by a record indicating that the transaction was committed.

##### Trigger-based replication

### Problems with Replication Lag

#### Reading Your Own Writes

*read-after-write consistency*
*cross-device read-after-write consistency*

#### Monotonic Reads

When you read data, you may see an old value; monotonic reads only means that if one user makes several reads in sequence, they will not see time go backward— i.e., they will not read older data after having previously read newer data.

One way of achieving monotonic reads is to make sure that each user always makes their reads from the same replica.

#### Consistent Prefix Reads

This guarantee says that if a sequence of writes happens in a certain order, then anyone reading those writes will see them appear in the same order.

#### Solutions for Replication Lag


### Multi-Leader Replication

#### Use Cases for Multi-Leader Replication

##### Multi-datacenter operation

Although multi-leader replication has advantages, it also has a big downside: the same data may be concurrently modified in two different datacenters, and those write conflicts must be resolved.

##### Clients with offline operation

Another situation in which multi-leader replication is appropriate is if you have an application that needs to continue to work while it is disconnected from the internet.

In this case, every device has a local database that acts as a leader (it accepts write requests), and there is an asynchronous multi-leader replication process (sync) between the replicas of your calendar on all of your devices. The replication lag may be hours or even days, depending on when you have internet access available.

There are tools that aim to make this kind of multi-leader configuration easier. For example, CouchDB is designed for this mode of operation.

##### Collaborative editing

*Real-time collaborative editing* applications allow several people to edit a document simultaneously. 

We don’t usually think of collaborative editing as a database replication problem, but it has a lot in common with the previously mentioned offline editing use case. 

#### Handling Write Conflicts

##### Synchronous versus asynchronous conflict detection

In a single-leader database, the second writer will either block and wait for the first write to complete, or abort the second write transaction, forcing the user to retry the write. On the other hand, in a multi-leader setup, both writes are successful, and the conflict is only detected asynchronously at some later point in time. At that time, it may be too late to ask the user to resolve the conflict.

##### Conflict avoidance

The simplest strategy for dealing with conflicts is to avoid them: if the application can ensure that all writes for a particular record go through the same leader, then con‐ flicts cannot occur. 

##### Converging toward a consistent state

That is not acceptable—every replication scheme must ensure that the data is eventually the same in all replicas. Thus, the database must resolve the conflict in a *convergent* way, which means that all replicas must arrive at the same final value when all changes have been replicated.

There are various ways of achieving convergent conflict resolution:

- Give each write a unique ID (e.g., a timestamp, a long random number, a UUID, or a hash of the key and value), pick the write with the highest ID as the *winner*, and throw away the other writes. If a timestamp is used, this technique is known as *last write wins* (LWW). Although this approach is popular, it is dangerously prone to data loss.

- Give each replica a unique ID, and let writes that originated at a higher- numbered replica always take precedence over writes that originated at a lower- numbered replica. This approach also implies data loss.

- Somehow merge the values together—e.g., order them alphabetically and then concatenate them.

- Record the conflict in an explicit data structure that preserves all information, and write application code that resolves the conflict at some later time.

##### Custom conflict resolution logic

That code may be executed on write or on read:

On write: As soon as the database system detects a conflict in the log of replicated changes, it calls the conflict handler.

On read: When a conflict is detected, all the conflicting writes are stored. The next time the data is read, these multiple versions of the data are returned to the application. The application may prompt the user or automatically resolve the conflict, and write the result back to the database. 

There has been some interesting research into automatically resolving conflicts caused by concurrent data modifications. A few lines of research are worth mentioning:

- *Conflict-free replicated datatypes* (CRDTs) are a family of data structures for sets, maps, ordered lists, counters, etc. that can be concurrently edited by multiple users, and which automatically resolve conflicts in sensible ways. Some CRDTs have been implemented in Riak 2.0.

- *Mergeable persistent data structures* track history explicitly, similarly to the Git version control system, and use a three-way merge function (whereas CRDTs use two-way merges).

- *Operational transformation* is the conflict resolution algorithm behind col‐ laborative editing applications such as Etherpad and Google Docs. It was designed particularly for concurrent editing of an ordered list of items, such as the list of characters that constitute a text document.

##### What is a conflict?

#### Multi-Leader Replication Topologies

A problem with *circular and star topologies* is that if just one node fails, it can inter‐ rupt the flow of replication messages between other nodes, causing them to be unable to communicate until the node is fixed.

On the other hand, *all-to-all topologies* can have issues too. In particular, some network links may be faster than others (e.g., due to network congestion), with the result that some replication messages may “overtake” others.

### Leaderless Replication

It once again became a fashiona‐ ble architecture for databases after Amazon used it for its in-house Dynamo system. Riak, Cassandra, and Voldemort are open source datastores with leaderless replication models inspired by Dynamo, so this kind of database is also known as Dynamo-style.

#### Writing to the Database When a Node Is Down

##### Read repair and anti-entropy

Two mechanisms are often used in Dynamo-style datastores:

- Read repair
- Anti-entropy process

Not all systems implement both of these; for example, Voldemort currently does not have an anti-entropy process. Note that without an anti-entropy process, values that are rarely read may be missing from some replicas and thus have reduced durability, because read repair is only performed when a value is read by the application.

##### Quorums for reading and writing

More generally, if there are n replicas, every write must be confirmed by w nodes to be considered successful, and we must query at least r nodes for each read. (In our example, n = 3, w = 2, r = 2.)

You can think of r and w as the minimum number of votes required for the read or write to be valid.

#### Limitations of Quorum Consistency

However, even with w + r > n, there are likely to be edge cases where stale values are returned. These depend on the implementation, but possible scenarios include:

- If a sloppy quorum is used, the w writes may end up on different nodes than the r reads, so there is no longer a guaranteed overlap between the r nodes and the w nodes.

- If two writes occur concurrently, it is not clear which one happened first. In this case, the only safe solution is to merge the concurrent writes.

- If a write happens concurrently with a read, the write may be reflected on only some of the replicas. In this case, it’s undetermined whether the read returns the old or the new value.

- If a write succeeded on some replicas but failed on others (for example because the disks on some nodes are full), and overall succeeded on fewer than w replicas, it is not rolled back on the replicas where it succeeded. This means that if a write was reported as failed, subsequent reads may or may not return the value from that write.

- If a node carrying a new value fails, and its data is restored from a replica carry‐ ing an old value, the number of replicas storing the new value may fall below w, breaking the quorum condition.

- Even if everything is working correctly, there are edge cases in which you can get unlucky with the timing, as we shall see in “Linearizability and quorums” on page 334.


In particular, you usually do not get the guarantees discussed in “Problems with Rep‐ lication Lag” on page 161 (reading your writes, monotonic reads, or consistent prefix reads), so the previously mentioned anomalies can occur in applications. Stronger guarantees generally require transactions or consensus. 

##### Monitoring staleness

#### Sloppy Quorums and Hinted Handoff



...
