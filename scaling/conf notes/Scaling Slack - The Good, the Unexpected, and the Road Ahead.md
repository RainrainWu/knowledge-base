# Scaling Slack - The Good, the Unexpected, and the Road Ahead
> [Original Video](https://www.youtube.com/watch?v=_M-oHxknfnI)

## Metrics
### 2016
- 4M DAU
- 2.5M peak simultaneous connected Avg 10 hrs/day
- \>10K active user in the largest org
- most system is >10 years tech
  - async job queue
  - server push update for synchronization via websocket
  - data sharing and routing
### 2018
- \>8M DAU (2x)
- \>7M peak simultaneous connected Avg 10 hrs/day (3x)
- \>125K active user in the largest org (10x)
- embrace complexity where needed to solve hardest problems

## Recurring Issues
- Large organization indicated large metadata while booting up client model
- Thundering herds: Load to connect >> Load in steady state
- Hot spots: overwhelm database hosts (mains and shards) and other system
- Herd of pets: manual operation to replace specific servers
- Cross workspace chcannel: need to change assumptions about partitioning

## Solution
### Thinner client model
- lazy load improves the performance on initial connect
- flannel cache can responds to >1M QPS, but need to take care of cache coherence

### Fine-Grained DB Sharding
- Adopt Vitess for building MySQL cluster
  - Flexible sharding policy
  - Single master
  - Failover automatically
  - Resharding workflows
- Migrating to channel-sharded/user-sharded from workspace-sharded data model helps mitigate hot spot for large teams.

### Service Decomposition
- Scale out channel server cluster, most of time it is the bottleneck

### From Herd of Pets to Service Mesh
- Evolved from hand-configured server to a discovery mesh (by adopting Consul)
  - self-registration and auto cluster repair
  - service ownership and on-call rotation

### Legacy Services
- Almost all of 2016 services are still in production
- Need to support legacy clients and integrations
- Data migrations need application changes takes time

