# Scaling Push Messaging for Millions of Devices @Netflix
> [Original Video](https://www.youtube.com/watch?v=6w6E_B55p0E&list=WL&index=43)

# Why need push?
- Time spent in Netflix application: picking movie > watching movie

# Zuul Push Architecture
## Zuul Push Server
- Adopt async io ro deal with C10K challenge

## Push Registry
- low read latency
- record expiry
- sharding
- replication

### solutions
- redis
- cassandra
- AWS dynamoDB
- Finally use Dynomite: Redis + auto sharding + read/write quorum + cross-region replication

## Message Processing
- Adopt Kafka to decouple sender and reciever.
  - different queus for different priorities

## Operating Zuul Push
- Long lived stable connections make Zuul push server stateful.
  - great for client efficiency
  - terrible for quick deploy/rollback
  - tear down connections periodically
  - to avoid thundering herd issue, randomize each connection's lifetime
- Most connections are idle
  - Goldilocks strategy (find the sweet spot of workloads via testing)
- More number of smaller servers >> few BIG servers
- How to auto-scale?
  - RPS and CPU metrics are not reliable since most of connection are idle
  - trigger scaling process based on the number of open connection
- AWS ELB cannot proxy websockets
  - Run ELB as a TCP loadbalancer

## If you build it, they will push
- On-demand diagnostics: push message ask the client to upload their state
  - maybe remote recovery after diagnostics
  - but still notify the users if diagnostics does not help

