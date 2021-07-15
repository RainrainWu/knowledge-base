# Anti-Entropy Using CRDTs on HA Datastores @Netflix
> [Original Video](https://www.youtube.com/watch?v=JirCwLXlH_c)

## Context
### Timeline
- 2011 Cassandra adoption from oracle for native HA
- 2013 Multi region storagement
- 2016 Dynomite
- 2019 dynomite w/ CRDTs

### Dynomite Overview
- Global replication
- HA
- Shared nothing
- Auto-sharding
- Linear scale
- Pluggable datastores
- Multiple qurom level
- Supports datastore API

## Entropy in the system
- Cannot obtain a consistent value based on the consensus of qurom due to unknow network partition.
  - replicas will go out of sync in a long run

### Achieve anti-entropy
- Last writer wins
  - clock skew exists in all nodes
- Vector clock (invremental version)
  - not allow concurrent write
- CRDT (conflict free replicated data type)

### CRDTs
- Features
  - Associated
    - (X + Y) + Z = X + (Y + Z)
  - Commutative
    - X + Y = Y + X
  - Idempotent
    - X + X = X
- Operations
  - Update local value
  - Merge with replicas
- Dynomites' implementation
  - Update while write
  - Merge while repair
    - Merge on read path while read repair
