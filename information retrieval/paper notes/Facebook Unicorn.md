# Unicorn: A System for Searching the Social Graph

> [Original Paper](https://dl.acm.org/doi/pdf/10.14778/2536222.2536239)

## Introduction
### Goal
-  Quickly and scalably search all basic structured information on the social graph.
-  Perform complex set operations on the results.

### Context
-  The primary backend system for Facebook Graph Search.
-  Billions of queries per day.
-  Response latencies less than a few hundred milliseconds.
-  Tens of billions of nodes and trillions of edges at scale.
-  Realtime updates for all edges and nodes.

## The Social Graph
- It consists of nodes signifying people and things; and edges representing a relationship between two nodes.
- Some edges are directional while others are symmetric.
- There are many thousands of edge-types.
- It is quite sparse: a typical node will have less than one thousand edges connecting it to other nodes.
  - The average user has approximately 130 friends.
  - The most popular pages and applications have tens of millions of edges, but these pages represent a tiny fraction of the total number of entities in the graph.

## Data Model
- An inverted index service that implements an adjacency list service, contains a sorted list of hits, which are (DocId, HitData) pairs.
  - Hits are sorted first by sort-key (highest first) and secondly by id (lowest first).
- One application of HitData is for storing extra data useful for filtering results.
- Chose to partition by result-id because we wanted the system to remain available in the event of a dead machine or network partition.
  - A further advantage of document-sharding instead of term-sharding is that most set operations can be done at the index server (leaf) level instead of being performed higher in the execution stack.

## API AND QUERY LANGUAGE
- The client passes the Thrift request to a library that handles sending the query to the correct Unicorn cluster.
  - Sent to a Unicorn cluster that is in close geographical proximity to the client if possible.
- Supports And and Or operators.
  - Find all female friends of Jon Jones would issue the query (and friend:5 gender:1).
  - Find all friends of Jon Jones or Lea Lin by issuing the query (or friend:5 friend:6).
  - Find female friends of Jon Jones who are not friends of Lea Lin by using (difference (and friend:5 gender:1) friend:6).

## ARCHITECTURE
