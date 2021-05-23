# Scaling Instagram Infrastructure

###### tags: `QCon`, `Django`, `PostgreSQL`, `Cassandra`

## Tech metrics

- 400M DAU
- 4+B likes per day
- 100M vedio/photo uploads
- Top account: 110M followers

## Scale dimentions

- Scale out
Build infras for more server/hardwares.
- Scale up
讓每個 Server 有更強的能力
- Scale the dev team
有更強的開發能力維持快節奏開發

### Scale Out
#### Storage

- Read Write Isolation for high QPS, and also use batch operation
- Cassandra does not has master role, focus on consistency
- Memcache does not has global consistency
- Denormalize column or table

#### Computing

- Don't count the servers,  make the servers count
- Use as few CPU instruction as possible, and bechmark while adding new feature
- Python cprofile

### Scale up

#### Memory
- Reduce code for less memory use
    - Run in optimized mode (-O)
    - Remove dead code
- Share more
    - Share config and artifacts into shared memory
    - Disable garbage collection

#### Async

- async each independent tasks

#### Runtime

- Cython is not debugging friendly, keep looking for
    - Faster python runtime
    - Async framework
    - Better memory analysis

### Scale the dev team

- Use master branch only
    - Keep master branch work through CI
    - Benchmarks all feature at the same time
    - Control feature realease through flag instead of feature branch
