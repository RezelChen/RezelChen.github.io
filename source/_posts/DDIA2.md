---
title: Note for DDIA in Chapter 2
date: 2020-07-24 18:54:40
tags: note
---

## Data Models and Query Languages

### Relational Model Versus Document Model

#### Many-to-One and Many-to-Many Relationships

Removing such duplication is the key idea behind *normalization* in databases.

As a rule of thumb, if you’re duplicating values that could be stored in just one place, the schema is not normalized.

#### Are Document Databases Repeating History?

##### The network model

##### The relational model

##### Comparison to document databases

#### Relational Versus Document Databases Today

##### Which data model leads to simpler application code?

##### Schema flexibility in the document model
A more accurate term is schema-on-read (the structure of the data is implicit, and only interpreted when the data is read), in contrast with schema-on-write (the traditional approach of relational databases, where the schema is explicit and the database ensures all written data conforms to it)

##### Data locality for queries

##### Convergence of document and relational databases

A hybrid of the relational and document models is a good route for databases to take in the future.

### Query Languages for Data

A declarative query language is attractive because it is typically more concise and eas‐ ier to work with than an imperative API. But more importantly, it also hides imple‐ mentation details of the database engine, which makes it possible for the database system to introduce performance improvements without requiring any changes to queries.

#### Declarative Queries on the Web

#### MapReduce Querying

MapReduce is a fairly low-level programming model for distributed execution on a cluster of machines. Higher-level query languages like SQL can be implemented as a pipeline of MapReduce operations but there are also many dis‐ tributed implementations of SQL that don’t use MapReduce.

Note there is nothing in SQL that constrains it to running on a single machine, and MapReduce doesn’t have a monopoly on distributed query execution.

### Graph-Like Data Models

In the examples just given, all the vertices in a graph represent the same kind of thing (people, web pages, or road junctions, respectively). However, graphs are not limited to such homogeneous data: an equally powerful use of graphs is to provide a consis‐ tent way of storing completely different types of objects in a single datastore.

#### Property Graphs

properties of vertex:
- A unique identifier
- A set of outgoing edges
- A set of incoming edges
- A collection of properties (key-value pairs)

properties of edge:
- A unique identifier
- The vertex at which the edge starts (the tail vertex)
- The vertex at which the edge ends (the head vertex)
- A label to describe the kind of relationship between the two vertices
- A collection of properties (key-value pairs)

You can think of a graph store as consisting of two relational tables, one for vertices and one for edges

#### The Cypher Query Language
...