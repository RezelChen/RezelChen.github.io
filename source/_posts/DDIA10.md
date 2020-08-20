---
title: Note for DDIA in Chapter 10
date: 2020-08-18 18:23:00
tags: note
---

## Batch Processing

Let’s distinguish three different types of systems:

- Services (online systems)
- Batch processing systems (offline systems)
- Stream processing systems (near-real-time systems)

### Batch Processing with Unix Tools

#### Simple Log Analysis

##### Chain of commands versus custom program

##### Sorting versus in-memory aggregation

#### The Unix Philosophy

The philosophy was described in 1978 as follows:

1. Make each program do one thing well. 

2. Expect the output of every program to become the input to another, as yet unknown, program.

3. Design and build software, even operating systems, to be tried early, ideally within weeks. Don’t hesitate to throw away the clumsy parts and rebuild them.

4. Use tools in preference to unskilled help to lighten a programming task, even if you have to detour to build the tools and expect to throw some of them out after you’ve finished using them.

##### A uniform interface

If you expect the output of one program to become the input to another program, that means those programs must use the same data format -— in other words, a compatible interface. If you want to be able to connect *any* program’s output to any pro‐ gram’s input, that means that *all* programs must use the same input/output interface.

In Unix, that interface is a file (or, more precisely, a file descriptor).

Today it’s an exception, not the norm, to have programs that work together as smoothly as Unix tools do.

##### Separation of logic and wiring

Another characteristic feature of Unix tools is their use of standard input (`stdin`) and standard output (`stdout`). 

You can even write your own programs and combine them with the tools provided by the operating system. Your program just needs to read input from `stdin` and write output to `stdout`, and it can participate in data processing pipelines. 

However, there are limits to what you can do with `stdin` and `stdout`. Programs that need multiple inputs or outputs are possible but tricky.

##### Transparency and experimentation

Part of what makes Unix tools so successful is that they make it quite easy to see what is going on:

- The input files to Unix commands are normally treated as immutable. This means you can run the commands as often as you want, trying various command-line options, without damaging the input files.

- You can end the pipeline at any point, pipe the output into less, and look at it to see if it has the expected form. This ability to inspect is great for debugging.

- You can write the output of one pipeline stage to a file and use that file as input to the next stage. This allows you to restart the later stage without rerunning the entire pipeline.

However, the biggest limitation of Unix tools is that they run only on a single machine—and that’s where tools like Hadoop come in.

### MapReduce and Distributed Filesystems

A single MapReduce job is comparable to a single Unix process: it takes one or more inputs and produces one or more outputs.

As with most Unix tools, running a MapReduce job normally does not modify the input and does not have any side effects other than producing the output. 

#### MapReduce Job Execution

The pattern of data processing in MapReduce:

1. Read a set of input files, and break it up into *records*. 

2. Call the mapper function to extract a key and value from each input record. 

3. Sort all of the key-value pairs by key.

4. Call the reducer function to iterate over the sorted key-value pairs. If there are multiple occurrences of the same key, the sorting has made them adjacent in the list, so it is easy to combine those values without having to keep a lot of state in memory. 

Viewed like this, the role of the mapper is to prepare the data by putting it into a form that is suitable for sorting, and the role of the reducer is to process the data that has been sorted.

##### Distributed execution of MapReduce

The MapReduce scheduler (not shown in the diagram) tries to run each mapper on one of the machines that stores a replica of the input file, provided that machine has enough spare RAM and CPU resources to run the map task. This principle is known as *putting the computation near the data*: it saves copying the input file over the network, reducing network load and increasing locality.

The reduce side of the computation is also partitioned. While the number of map tasks is determined by the number of input file blocks, the number of reduce tasks is configured by the job author (it can be different from the number of map tasks).

The key-value pairs must be sorted, but the dataset is likely too large to be sorted with a conventional sorting algorithm on a single machine. Instead, the sorting is per‐ formed in stages. First, each map task partitions its output by reducer, based on the hash of the key. Each of these partitions is written to a sorted file on the mapper’s local disk, using a technique similar to what we discussed in “SSTables and LSM- Trees” on page 76.

The process of partitioning by reducer, sorting, and copying data partitions from mappers to reducers is known as the *shuffle*.

The reduce task takes the files from the mappers and merges them together, preserv‐ ing the sort order. 

The reducer is called with a key and an iterator that incrementally scans over all records with the same key.

##### MapReduce workflows

It is very common for MapReduce jobs to be chained together into workflows, such that the output of one job becomes the input to the next job. 

A batch job’s output is only considered valid when the job has completed successfully. Therefore, one job in a workflow can only start when the prior jobs -— that is, the jobs that produce its input directories —- have completed successfully.

These schedulers also have management features that are useful when maintaining a large collection of batch jobs.

#### Reduce-Side Joins and Grouping

When we talk about joins in the context of batch processing, we mean resolving all occurrences of some association within a dataset. For example, we assume that a job is processing the data for all users simultaneously, not merely looking up the data for one particular user (which would be done far more efficiently with an index).

##### Example: analysis of user activity events

##### Sort-merge joins

##### Bringing related data together in the same place

In a sort-merge join, the mappers and the sorting process make sure that all the nec‐ essary data to perform the join operation for a particular user ID is brought together in the same place: a single call to the reducer. 

One way of looking at this architecture is that mappers “send messages” to the reduc‐ ers. When a mapper emits a key-value pair, the key acts like the destination address to which the value should be delivered. Even though the key is just an arbitrary string (not an actual network address like an IP address and port number), it behaves like an address: all key-value pairs with the same key will be delivered to the same destination (a call to the reducer).

##### GROUP BY

##### Handling skew

The pattern of “bringing all records with the same key to the same place” breaks down if there is a very large amount of data related to a single key. 

When performing the actual join, the mappers send any records relating to a hot key to one of several reducers, chosen at random (in contrast to conventional MapReduce, which chooses a reducer deterministically based on a hash of the key). For the other input to the join, records relating to the hot key need to be replicated to all reducers handling that key.


#### Map-Side Joins

The join algorithms described in the last section perform the actual join logic in the reducers, and are hence known as reduce-side joins. The mappers take the role of preparing the input data: extracting the key and value from each input record, assigning the key-value pairs to a reducer partition, and sorting by key.

##### Broadcast hash joins

The simplest way of performing a map-side join applies in the case where a large dataset is joined with a small dataset. In particular, the small dataset needs to be small enough that it can be loaded entirely into memory in each of the mappers.

This simple but effective algorithm is called a *broadcast hash join*: the word broadcast reflects the fact that each mapper for a partition of the large input reads the entirety of the small input (so the small input is effectively “broadcast” to all partitions of the large input), and the word hash reflects its use of a hash table. 

##### Partitioned hash joins

If the inputs to the map-side join are partitioned in the same way, then the hash join approach can be applied to each partition independently. 

This approach only works if both of the join’s inputs have the same number of partitions, with records assigned to partitions based on the same key and the same hash function. If the inputs are generated by prior MapReduce jobs that already perform this grouping, then this can be a reasonable assumption to make.

##### Map-side merge joins

Another variant of a map-side join applies if the input datasets are not only parti‐ tioned in the same way, but also *sorted* based on the same key. 

If a map-side merge join is possible, it probably means that prior MapReduce jobs brought the input datasets into this partitioned and sorted form in the first place. In principle, this join could have been performed in the reduce stage of the prior job. However, it may still be appropriate to perform the merge join in a separate map- only job, for example if the partitioned and sorted datasets are also needed for other purposes besides this particular join.

##### MapReduce workflows with map-side joins

When the output of a MapReduce join is consumed by downstream jobs, the choice of map-side or reduce-side join affects the structure of the output. The output of a reduce-side join is partitioned and sorted by the join key, whereas the output of a map-side join is partitioned and sorted in the same way as the large input.

#### The Output of Batch Workflows

The output of a batch process is often not a report, but some other kind of structure.

##### Building search indexes

If you need to perform a full-text search over a fixed set of documents, then a batch process is a very effective way of building the indexes: the mappers partition the set of documents as needed, each reducer builds the index for its partition, and the index files are written to the distributed filesystem. 

##### Key-value stores as batch process output

Another common use for batch processing is to build machine learning systems such as classifiers (e.g., spam filters, anomaly detection, image recognition) and recommendation systems (e.g., people you may know, products you may be interested in, or related searches).

A much better solution is to build a brand-new database *inside* the batch job and write it as files to the job’s output directory in the distributed filesystem, just like the search indexes in the last section. Those data files are then immutable once written, and can be loaded in bulk into servers that handle read-only queries. Various key- value stores support building database files in MapReduce jobs, including Voldemort, Terrapin, ElephantDB, and HBase bulk loading.

##### Philosophy of batch process outputs

#### Comparing Hadoop to Distributed Databases

##### Diversity of storage
...
### Beyond MapReduce

In response to the difficulty of using MapReduce directly, various higher-level pro‐ gramming models (Pig, Hive, Cascading, Crunch) were created as abstractions on top of MapReduce.

In the rest of this chapter, we will look at some of those alternatives for batch processing. In Chapter 11 we will move to stream processing, which can be regarded as another way of speeding up batch processing.

#### Materialization of Intermediate State

Pipes do not fully materialize the intermediate state, but instead *stream* the output to the input incrementally, using only a small in-memory buffer.

MapReduce’s approach of fully materializing intermediate state has downsides com‐ pared to Unix pipes:

- A MapReduce job can only start when all tasks in the preceding jobs have completed, whereas processes connected by a Unix pipe are started at the same time, with output being consumed as soon as it is produced.

- Mappers are often redundant: they just read back the same file that was just writ‐ ten by a reducer, and prepare it for the next stage of partitioning and sorting. 

- Storing intermediate state in a distributed filesystem means those files are repli‐ cated across several nodes, which is often overkill for such temporary data.

##### Dataflow engines

In order to fix these problems with MapReduce, several new execution engines for distributed batch computations were developed, the most well known of which are Spark, Tez, and Flink. They handle an entire workflow as one job, rather than breaking it up into independent subjobs.

Since they explicitly model the flow of data through several processing stages, these systems are known as *dataflow engines*.

You can use dataflow engines to implement the same computations as MapReduce workflows, and they usually execute significantly faster due to the optimizations described here. 

##### Fault tolerance

Spark, Flink, and Tez avoid writing intermediate state to HDFS, so they take a different approach to tolerating faults: if a machine fails and the intermediate state on that machine is lost, it is recomputed from other data that is still available (a prior inter‐ mediary stage if possible, or otherwise the original input data, which is normally on HDFS).

Recovering from faults by recomputing data is not always the right answer: if the intermediate data is much smaller than the source data, or if the computation is very CPU-intensive, it is probably cheaper to materialize the intermediate data to files than to recompute it.

##### Discussion of materialization

#### Graphs and Iterative Processing
...
#### High-Level APIs and Languages

##### The move toward declarative query languages

An advantage of specifying joins as relational operators, compared to spelling out the code that performs the join, is that the framework can analyze the properties of the join inputs and automatically decide which of the aforementioned join algorithms would be most suitable for the task at hand.

By incorporating declarative aspects in their high-level APIs, and having query opti‐ mizers that can take advantage of them during execution, batch processing frameworks begin to look more like MPP databases (and can achieve comparable performance). At the same time, by having the extensibility of being able to run arbitrary code and read data in arbitrary formats, they retain their flexibility advantage.

##### Specialization for different domains

Also useful are spatial algorithms such as *k-nearest neighbors*, which searches for items that are close to a given item in some multi-dimensional space—a kind of simi‐ larity search. Approximate search is also important for genome analysis algorithms, which need to find strings that are similar but not identical.

### Summary
...
