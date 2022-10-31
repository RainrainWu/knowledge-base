# PostgreSQL

- [PostgreSQL](#postgresql)
  - [Indexes](#indexes)
    - [Built-in Indexes](#built-in-indexes)
  - [Reliability](#reliability)
    - [Snapshot](#snapshot)
    - [Write-Ahead-Log (WAL)](#write-ahead-log-wal)
  - [Clustering](#clustering)
  - [pgpool-II](#pgpool-ii)

## Indexes

### Built-in Indexes
> PostgreSQL provide several widely-used built-in indexes that allow users to pick the suitable one by case.

**B-Tree**
- Based on multi-way balanced search tree which support general comparison (e.g. `>=`, `<`, ...).
- [PostgreSQL Official Documents](https://www.postgresql.org/docs/current/btree-intro.html)

**GiST**
- Also a kind of balanced search tree but allows more flexible data type and compare methods (e.g. geometry overlap, distance based on latitude and longitude).
- [PostgreSQL Official Document](https://www.postgresql.org/docs/current/gist-intro.html)

**Space-partitioned Generalized Search Tree (SP-GiST)**
- More focus on abstract space partitioning and searching.
  - Quad-Tree for geo-spatial partition.
  - Trie based on zip-code or ip address.
- Most of the data here can lead to unbalance tree easily, which is not friendly to GiST.
- [PostgreSQL Official Document](https://www.postgresql.org/docs/current/spgist-intro.html)

**GIN**
- Focus on searching desired items which contained inside large composited value (e.g. JSON, array, text, ...).
- [PostgreSQL Official Document](https://www.postgresql.org/docs/current/gin-builtin-opclasses.html)

## Reliability

### Snapshot
- Creating snapshot by `CHECKPOINT` command periodically contributes to faster disaster recovery from system failure.

### Write-Ahead-Log (WAL)
- Record each command in append-only-file before execute it, which enable accurately point-in-time recovery by transaction.
- WAL is also the underlying mechanism for synchronization inside a cluster.

## Clustering
- Each postgresql instance has its own connection quota, which can improve the overall throughput directly.
- Synchronization can be an expensive operation sometimes, user should consider the trade-offs between consistency and performance.

**References**
- [What is streaming replication, and how can I set it up?](https://www.postgresql.fastware.com/postgresql-insider-ha-str-rep)
- [PostgreSQL High Availability using pgpool-II](https://www.postgresql.fastware.com/postgresql-insider-ha-pgpool-ii)

## pgpool-II
> pgpool_II act as a reversed-proxy in front of postgresql cluster, which has the following advantages:

**Load Balancing**
- Route the read operation to available replica instances.

**Connection Pooling**
- Managed connection pool to save the cost on re-connecting.

**Replication**
- Maintain the replicating process within cluster (write-ahead-log was recommended officially), which is critical for high-availability design.

**Failover & Recovery**
- pgpool-II monitor the instance via periodically heartbeat, failover can be triggered automatically once the primary instance is unavailable.
- Node can be recovered or added without downtime.

**Highly Available Design**
- pgpool-II itself can also be set up in a highly available cluster architecture.
- Be careful about the split-brain issue during the primary election.
