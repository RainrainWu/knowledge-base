# RDBMS Generals

- [RDBMS Generals](#rdbms-generals)
  - [ACID](#acid)
    - [Atomic](#atomic)
    - [Consistency](#consistency)
    - [Isolation](#isolation)
    - [Durability](#durability)
  - [Normalization](#normalization)
  - [Indexing](#indexing)
    - [B+Tree](#btree)
    - [Clustered Index (Index Organized Table)](#clustered-index-index-organized-table)
    - [Heap Table](#heap-table)
  - [Concurrency Control](#concurrency-control)
    - [Read Phenomena](#read-phenomena)
    - [Isolation Level](#isolation-level)
  - [Lock](#lock)
    - [Shared-Exclusive Lock (SX Lock)](#shared-exclusive-lock-sx-lock)
    - [Multi Version Concurrency Control (MVCC)](#multi-version-concurrency-control-mvcc)

## ACID
### Atomic
- A transaction is able to carry multiple sql commands, but it must be success completely or failed completely.
- Disaster recovery must be done by transaction.

### Consistency
- Database must transfer to a valid state from the previous valid state.
- The data inside must always follow the defined rules (e.g. constrains, cascade, triggers).

### Isolation
- Transactions executed concurrently should never affect each other, and only the data before the executed moment is visible.
- The eventual state of concurrently executed transactions must be the same as sequential execution.

### Durability
- The mutation of a transaction must be remained even in case of system failure, which means it has been records in non-volatile storages.

## Normalization
- Most of the times, we should deliver a schema satisfy at least 3NF for future maintainability.

**References**
- [What is Normalization in DBMS (SQL)? 1NF, 2NF, 3NF Example](https://www.guru99.com/database-normalization.html)

## Indexing

### B+Tree
- Primary key usually been used to build the main B+Tree.
- Read performance can be improved by maintaining additional secondary indexes, but the trade-offs are the write performance and storage space.
- Non-leaf pages usually referred frequently for data locating, and may always stay in the memory space.

### Clustered Index (Index Organized Table)
<img width="868" alt="截圖 2021-05-11 下午3 39 46" src="https://user-images.githubusercontent.com/26277801/117777384-2104aa80-b26f-11eb-805e-fde596ddd648.png">

- Data was stored directly inside the leaf nodes.

**Pros**
- Already sorted by primary key, physically continuously stored data performs excellent on range scan operations.

**Cons**
- All create operations will occurs on the right most node, may lead to poor performance due to lock issues.
- Storage size of each node was limited, there can be many merge/split nodes operation if we frequently insert/delete records.
- Split/merge nodes move the physical location of records, so secondary indexes can only store the primary key instead of physical location, and it required an additional read operation to obtain the data.

### Heap Table
<img width="925" alt="截圖 2021-05-11 下午4 13 34" src="https://user-images.githubusercontent.com/26277801/117782057-db96ac00-b273-11eb-87d2-9f30c0af2a8a.png">

- Leaf node store the physical location of the records instead of the data itself.

**Pros**
- Records can be distributed to the low load node to avoid poor performance due to hot node.
- The physical location of records should never be moved, and secondary indexes can refer to the address directly.

**Cons**
- Any kind of operation will lead to arbitrary disk read, which have to load many data page into memory (heavy disk I/O and memory utilization).

## Concurrency Control

- Database will tend to execute as many transactions as possible concurrently for better performance.
- To strike a balance between performance and consistency, user can have choices between different levels of isolation depends on the requirements.

### Read Phenomena
- Transactions may affect each other depends on different isolation levels.

|Phenomenon|Description|
|-|-|
|Dirty Read|Uncommitted data is visible to other transaction.|
|Non-repeatable Read|SQL commands within the same transactions may obtain different versions of the same record due to other transactions.|
|Phantom Read|SQL commands within the same transactions may obtain different result of the same aggregation due to other transactions.|
|Serialization Anomaly|The eventual state of database may be different due to the order of transaction execution.|

### Isolation Level
- There are 4 isolation levels to handle the phenomena above.

|Isolation Level|Dirty Read|Nonrepeatable Read|Phantom Read|Serialization Anomaly|
|-|-|-|-|-|
|Read uncommitted|Allowed|Possible|Possible|Possible|
|Read committed|Not possible|Possible|Possible|Possible|
|Repeatable read|Not possible|Not possible|Allowed|Possible|
|Serializable|Not possible|Not possible|Not possible|Not possible|

## Lock
- Locks were been used to achieve the permission of operation for the isolation between transactions.

### Shared-Exclusive Lock (SX Lock)
- Each record maintains its own keys, S key for read operation and can have multiple instances, X key for write operation which only have 1 instance.
- X lock can only be lend if no other S/X keys held by other transactions.
- No S/X key can be lend before an lent X key was remanded.

**Pros**
- Resource-effective approach (compares to MVCC).

**Cons**
- Complicated policy with more lock instances which may cause higher possibility of race condition.

### Multi Version Concurrency Control (MVCC)
- Each record have different version of its data and the old versions can be cleaned up.
- The visibility of data was limited by the moment of execution of each transaction (newer versions are invisible.)

**Pros**
- No read lock needed, latest data for each transaction is always available, which leads to better performance for heavy read scenarios.

**Cons**
- Multiple versions management of each record requires heavy Disk I/O and CPU utilization.
- Cleaned up routine for old versions data leads to background workloads.
