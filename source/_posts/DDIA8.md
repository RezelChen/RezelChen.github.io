---
title: Note for DDIA in Chapter 8
date: 2020-08-13 19:53:00
tags: note
---

## The Trouble with Distributed Systems


### Faults and Partial Failures

An individual computer with good software is usually either fully functional or entirely broken, but not something in between.

This is a deliberate choice in the design of computers: if an internal fault occurs, we prefer a computer to crash completely rather than returning a wrong result, because wrong results are difficult and confusing to deal with. 

When you are writing software that runs on several computers, connected by a net‐ work, the situation is fundamentally different. 

This nondeterminism and possibility of partial failures is what makes distributed sys‐ tems hard to work with.

#### Cloud Computing and Supercomputing

If we want to make distributed systems work, we must accept the possibility of partial failure and build fault-tolerance mechanisms into the software. In other words, we need to build a reliable system from unreliable components.

### Unreliable Networks

#### Network Faults in Practice

#### Detecting Faults

#### Timeouts and Unbounded Delays

##### Network congestion and queueing

#### Synchronous Versus Asynchronous Networks

##### Can we not simply make network delays predictable?

### Unreliable Clocks

Clocks and time are important. Applications depend on clocks in various ways to answer questions like the following:

1. Has this request timed out yet?
2. What’s the 99th percentile response time of this service?
3. How many queries per second did this service handle on average in the last five minutes?
4. How long did the user spend on our site?
5. When was this article published?
6. At what date and time should the reminder email be sent?
7. When does this cache entry expire?
8. What is the timestamp on this error message in the log file?

#### Monotonic Versus Time-of-Day Clocks

##### Time-of-day clocks

A time-of-day clock does what you intuitively expect of a clock: it returns the current date and time according to some calendar (also known as wall-clock time). 

Time-of-day clocks are usually synchronized with NTP, which means that a time‐ stamp from one machine (ideally) means the same as a timestamp on another machine. 

##### Monotonic clocks

A monotonic clock is suitable for measuring a duration. The name comes from the fact that they are guaranteed to always move forward (whereas a time-of- day clock may jump back in time).

In a distributed system, using a monotonic clock for measuring elapsed time (e.g., timeouts) is usually fine, because it doesn’t assume any synchronization between dif‐ ferent nodes’ clocks and is not sensitive to slight inaccuracies of measurement.

#### Clock Synchronization and Accuracy

It is possible to achieve very good clock accuracy if you care about it sufficiently to invest significant resources. Such accuracy can be achieved using GPS receivers, the Precision Time Protocol (PTP), and careful deployment and monitoring. However, it requires significant effort and expertise, and there are plenty of ways clock synchronization can go wrong.

#### Relying on Synchronized Clocks
...
#### Process Pauses

##### Response time guarantees

##### Limiting the impact of garbage collection

A variant of this idea is to use the garbage collector only for short-lived objects (which are fast to collect) and to restart processes periodically, before they accumu‐ late enough long-lived objects to require a full GC of long-lived objects. One node can be restarted at a time, and traffic can be shifted away from the node before the planned restart, like in a rolling upgrade.

### Knowledge, Truth, and Lies

#### The Truth Is Defined by the Majority

Instead, many distributed algorithms rely on a quorum, that is, voting among the nodes: decisions require some minimum number of votes from several nodes in order to reduce the dependence on any one particular node.

##### The leader and the lock

Frequently, a system requires there to be only one of some thing. For example:

- Only one node is allowed to be the leader for a database partition, to avoid split
brain.

- Only one transaction or client is allowed to hold the lock for a particular resource
or object, to prevent concurrently writing to it and corrupting it.

- Only one user is allowed to register a particular username, because a username must uniquely identify a user.

##### Fencing tokens

Checking a token on the server side may seem like a downside, but it is arguably a good thing: it is unwise for a service to assume that its clients will always be well behaved, because the clients are often run by people whose priorities are very differ‐ ent from the priorities of the people running the service. Thus, it is a good idea for any service to protect itself from accidentally abusive clients.

#### Byzantine Faults
...
#### System Model and Reality

With regard to timing assumptions, three system models are in common use:

- Synchronous model: The synchronous model assumes bounded network delay, bounded process pau‐ ses, and bounded clock error. This does not imply exactly synchronized clocks or zero network delay; it just means you know that network delay, pauses, and clock drift will never exceed some fixed upper bound. 

- Partially synchronous model: Partial synchrony means that a system behaves like a synchronous system most of the time, but it sometimes exceeds the bounds for network delay, process pauses, and clock drift.

- Asynchronous model: In this model, an algorithm is not allowed to make any timing assumptions.

Moreover, besides timing issues, we have to consider node failures. The three most common system models for nodes are:

- Crash-stop faults: In the crash-stop model, an algorithm may assume that a node can fail in only one way, namely by crashing. This means that the node may suddenly stop responding at any moment, and thereafter that node is gone forever—it never comes back.

- Crash-recovery faults: We assume that nodes may crash at any moment, and perhaps start responding again after some unknown time. In the crash-recovery model, nodes are assumed to have stable storage (i.e., nonvolatile disk storage) that is preserved across crashes, while the in-memory state is assumed to be lost.

- Byzantine (arbitrary) faults: Nodes may do absolutely anything, including trying to trick and deceive other nodes, as described in the last section.

For modeling real systems, the partially synchronous model with crash-recovery faults is generally the most useful model. 

##### Correctness of an algorithm

##### Safety and liveness

*Safety* is often informally defined as *nothing bad happens*, and *liveness* as *something good eventually happens*.

The actual definitions of safety and liveness are precise and mathematical:

- If a safety property is violated, we can point at a particular point in time at which it was broken. After a safety property has been violated, the violation cannot be undone—the damage is already done.

- A liveness property works the other way round: it may not hold at some point in time, but there is always hope that it may be satisfied in the future.

##### Mapping system models to the real world

### Summary

...
