---
title: Note for DDIA in Chapter 3
date: 2020-07-24 23:05:00
tags: note
---

## Storage and Retrieval

We will examine two families of storage engines: *log-structured* storage engines, and *page-oriented* storage engines such as B-trees.

### Data Structures That Power Your Database

#### Hash Indexes

Segments are never modified after they have been written, so the merged segment is written to a new file. The merging and compaction of frozen segments can be done in a background thread, and while it is going on, we can still continue to serve read and write requests as normal, using the old segment files. After the merging process is complete, we switch read requests to using the new merged segment instead of the old segments — and then the old segment files can simply be deleted.

Each segment now has its own in-memory hash table, mapping keys to file offsets. 

An append-only design turns out to be good for several reasons:
- Appending and segment merging are sequential write operations, which are generally much faster than random writes, especially on magnetic spinning-disk hard drives.

- Concurrency and crash recovery are much simpler if segment files are append-only or immutable.

- Merging old segments avoids the problem of data files getting fragmented over time.

The hash table index also has limitations:
- The hash table must fit in memory, so if you have a very large number of keys, you’re out of luck.
- Range queries are not efficient. 

#### SSTables and LSM-Trees

SSTables have several big advantages over log segments with hash indexes:

- Merging segments is simple and efficient, even if the files are bigger than the available memory. The approach is like the one used in the *mergesort* algorithm

- In order to find a particular key in the file, you no longer need to keep an index of all the keys in memory. You still need an in-memory index to tell you the offsets for some of the keys, but it can be sparse: one key for every few kilobytes of segment file is sufficient, because a few kilobytes can be scanned very quickly.i

- Since read requests need to scan over several key-value pairs in the requested range anyway, it is possible to group those records into a block and compress it before writing it to disk. Each entry of the sparse in-memory index then points at the start of a compressed block.

##### Constructing and maintaining SSTables
- When a write comes in, add it to an in-memory balanced tree data structure (for
example, a red-black tree). This in-memory tree is sometimes called a memtable.

- When the memtable gets bigger than some threshold—typically a few megabytes—write it out to disk as an SSTable file. This can be done efficiently because the tree already maintains the key-value pairs sorted by key. The new SSTable file becomes the most recent segment of the database. While the SSTable is being written out to disk, writes can continue to a new memtable instance.

- In order to serve a read request, first try to find the key in the memtable, then in the most recent on-disk segment, then in the next-older segment, etc.

- From time to time, run a merging and compaction process in the background to combine segment files and to discard overwritten or deleted values.

This scheme works very well. It only suffers from one problem: if the database crashes, the most recent writes (which are in the memtable but not yet written out to disk) are lost. In order to avoid that problem, we can keep a separate log on disk to which every write is immediately appended, just like in the previous section. That log is not in sorted order, but that doesn’t matter, because its only purpose is to restore the memtable after a crash. Every time the memtable is written out to an SSTable, the corresponding log can be discarded.

##### Making an LSM-tree out of SSTables

##### Performance optimizations

The LSM-tree algorithm can be slow when looking up keys that do not exist in the database. In order to optimize this kind of access, storage engines often use additional *Bloom filters*.

There are also different strategies to determine the order and timing of how SSTables are compacted and merged. The most common options are *size-tiered* and *leveled* compaction.

#### B-Trees

Like SSTables, B-trees keep key-value pairs sorted by key, which allows efficient key- value lookups and range queries.

B-trees break the database down into fixed-size blocks or pages, traditionally 4 KB in size (sometimes bigger), and read or write one page at a time. 

Each page can be identified using an address or location, which allows one page to refer to another—similar to a pointer, but on disk instead of in memory. 

##### Making B-trees reliable

In order to make the database resilient to crashes, it is common for B-tree implemen‐ tations to include an additional data structure on disk: a write-ahead log (WAL, also known as a redo log). 

An additional complication of updating pages in place is that careful concurrency control is required if multiple threads are going to access the B-tree at the same time —otherwise a thread may see the tree in an inconsistent state. This is typically done by protecting the tree’s data structures with *latches* (lightweight locks).

##### B-tree optimizations

- Instead of overwriting pages and maintaining a WAL for crash recovery, some databases (like LMDB) use a copy-on-write scheme. A modified page is written to a different location, and a new version of the parent pages in the tree is created, pointing at the new location. 

- We can save space in pages by not storing the entire key, but abbreviating it.

- If a query needs to scan over a large part of the key range in sorted order, that page-by-page layout can be ineffi‐ cient, because a disk seek may be required for every page that is read. Many B- tree implementations therefore try to lay out the tree so that leaf pages appear in sequential order on disk. 

- Additional pointers have been added to the tree. For example, each leaf page may have references to its sibling pages to the left and right, which allows scanning keys in order without jumping back to parent pages.

- B-tree variants such as fractal trees borrow some log-structured ideas to reduce disk seeks (and they have nothing to do with fractals).

#### Comparing B-Trees and LSM-Trees

##### Advantages of LSM-trees

LSM-trees are typically able to sustain higher write throughput than B- trees.

LSM-trees can be compressed better, and thus often produce smaller files on disk than B-trees. 

##### Downsides of LSM-trees

A downside of log-structured storage is that the compaction process can sometimes interfere with the performance of ongoing reads and writes. 

Another issue with compaction arises at high write throughput: the disk’s finite write bandwidth needs to be shared between the initial write (logging and flushing a memtable to disk) and the compaction threads running in the background.

If write throughput is high and compaction is not configured carefully, it can happen that compaction cannot keep up with the rate of incoming writes.

An advantage of B-trees is that each key exists in exactly one place in the index, whereas a log-structured storage engine may have multiple copies of the same key in different segments. This aspect makes B-trees attractive in databases that want to offer strong transactional semantics: in many relational databases, transaction isola‐ tion is implemented using locks on ranges of keys, and in a B-tree index, those locks can be directly attached to the tree 

#### Other Indexing Structures

##### Storing values within the index

In some situations, the extra hop from the index to the heap file is too much of a performance penalty for reads, so it can be desirable to store the indexed row directly within an index. This is known as a *clustered index*.

##### Multi-column indexes
...

### Transaction Processing or Analytics?

online transaction processing(OLTP)
online analytic processing (OLAP)

#### Data Warehousing

The data warehouse con‐ tains a read-only copy of the data in all the various OLTP systems in the company. Data is extracted from OLTP databases (using either a periodic data dump or a con‐ tinuous stream of updates), transformed into an analysis-friendly schema, cleaned up, and then loaded into the data warehouse. This process of getting data into the warehouse is known as Extract–Transform–Load (ETL)

##### The divergence between OLTP databases and data warehouses

#### Stars and Snowflakes: Schemas for Analytics
...

### Column-Oriented Storage

...
