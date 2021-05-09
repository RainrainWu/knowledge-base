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
- Client queries are sent to a Unicorn top-aggregator, which dispatches the query to one rack-aggregator per rack.
  - The rack aggregators and top-aggregator are responsible for combining and truncating results from multiple index shards.
- The posting lists are generally shorter, with the 50th and 99th percentiles within a representative shard being 3 and 59 hits.
  -  Decoding speed is dominated by memory access time for very short posting lists, we get the greatest performance gains by optimizing for decoding the posting lists that consume more than a single CPU-cache line.

### Building Indices
- Built using the Hadoop framework.
- The raw data comes from regular scrapes of our databases that contain the relevant table columns.
- Many applications within Facebook require their data to be fresh with the latest up-to-the-minute updates to the social graph, we also support realtime updates.
- For a given update-type, which is known as a category, each index server keeps track of the timestamp of the latest update it has received for that category.
- When a new index server becomes available after a network partition or machine replacement, the scribe tailer can query the index server for its latest update timestamps for each category and then send it all missing mutations.

## TYPEAHEAD
- Typeahead enables Facebook users to find other users by typing the first few characters of the person’s name.
- For example, if a user is typing in the name of “Jon Jones” the typeahead backend would sequentially receive queries for “J”, “Jo”, “Jon”, “Jon ”, “Jon J”, etc.

### Social Relevance
- One problem with the approach described above is that it makes no provision for social relevance.
  - A query for “Jon” would not be likely to select people named “Jon” who are in the user’s circle of friends.

#### WeakAnd
- The WeakAnd operator is a modification of And that allows operands to be missing from some fraction of the results within an index shard.
  - For these optional terms, clients can specify an optional count or optional weight that allows a term to be missing from a certain absolute number or fraction of results, respectively.
  - If the optional count for a term is non-zero , then this term is not required to yield a particular hit, but the index server decrements the optional count for each such non-matching hit.
  - Once the optional count for a particular term reaches zero, the term becomes a required term.

#### StrongOr
- A modification of Or that requires certain operands to be present in some fraction of the matches.
  - Useful for enforcing diversity in a result set.

#### Scoring
- For storing per-entity metadata, Unicorn provides a forward index, which is simply a map of id to a blob that contains metadata for the id.
- The forward index for an index shard only contains entries for the ids that reside on that shard.

#### Additional Entities
- Typeahead can also be used to search for other entitytypes in addition to people.
  - Split into multiple entity-type specific tiers—or verticals—and modified the top-aggregator to query and combine results from multiple verticals.
  - Edges of the same result-type are placed in the same vertical, so the posting lists for friend edges and likers edges reside in the same vertical since they both yield user ids.
  - By partitioning ids by vertical, verticals can be replicated as necessary to serve the query load specific to that entity type.

## GRAPH SEARCH
- There are interesting graph results that are more than one edge away from the source nodes.
  - Requires supporting queries that take more than one round-trip between the index server and the top-aggregator.
  - Take a pre-specified edge-type from the client and combine it with result ids from a round of execution to generate and execute a new query.
- Build a general-purpose, online system for users to find entities in the social graph that matched a set of user-defined constraints.

### Apply
- Allows clients to query for a set of ids and then use those ids to construct and execute a new query.
  - The network latency saved by doing extra processing within the Unicorn cluster itself.
  - The ability write the aggregation and processing of the extra steps in a high performance language.
- Think of Apply as analogous to Join because it can leverage aggregate data and metadata to execute queries more efficiently than clients themselves.

#### Example: Friends-of-Friends
- While each user has 130 friends on average, they have approximately 48000 friends-of-friends.
  - Even assuming only 4 bytes per posting list entry, this would still take over 178TB to store the posting lists for 1 billion users, which would require hundreds of machines.
  - Here, we only have terms for friend:. Given the same assumptions, this corpus would take up 484GB and would fit comfortably on a handful of machines.
- Rank them by the number of terms that matched each result.

#### Example: Restaurants
- The inner query of the apply operator would have millions of results, which is too many to return to the top-aggregator.
- Good inner result ranking solves the vast majority of truncation problems, although this is an area of active monitoring and development.

### Extract
- Created as a way to tackle a common use-case: Say a client wishes to look up “People tagged in photos of Jon Jones”.
  - Billions of “one-to-few” mappings, it makes better sense to store the result ids in the forward index of the secondary vertical and do the lookup inline.

## LINEAGE
- Certain graph edges cannot be shown to all users but rather only to users who are friends with or in the same network as a particular person.
  - Unicorn itself does not have privacy information incorporated into its index. 
  - Give callers all the relevant data concerning how a particular result was generated in Unicorn so that the caller can make a proper privacy check on the result.
- First, we did not want our system to provide the strict consistency and durability guarantees that would be needed for a full privacy solution.
  -  If a user “un-friends” another user, the first user’s friends-only content immediately and irrevocably must become invisible to the second user.
- Facebook has organically accumulated a massive set of complex, privacy-related business logic in its frontend.
- A string of metadata is attached to each search result to describe its lineage.

## HANDLING FAILURES AND SCALE
- Sharding by id provides some degree of “passive” safety, since serving incomplete results for a term is strongly preferable to serving empty results.
- Rack aggregators know how to forward requests to a shard copy in another rack in the event that a shard becomes unavailable.
- It is possible to build a custom, smaller subset of a particular vertical to provide extra replication for these hot term families.

## PERFORMANCE EVALUATION
- Evaluated the query “People who like Computer Science”.
  - There are over 6M likers of computer science on Facebook.
- Next, we build an Apply on top of this base query by finding the friends of these individuals.
  - How performance is affected as the inner truncation limit is increased from 10 to 100k.
- Relative deviation increases gradually as the truncation limit increases.

## RELATED WORK
- Unicorn seeks to handle a finite number of well understood edges and scale to trillions of edges.
- SPARQL engines intend to handle arbitrary graph structure and complex queries, and scale to tens of millions of edges. 
