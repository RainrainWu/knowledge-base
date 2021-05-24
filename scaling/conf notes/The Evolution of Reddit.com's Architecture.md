# The Evolution of Reddit.com's Architecture
> [Original Video](https://www.youtube.com/watch?v=nUcO7n4hek4&t=3s)

## Metrics
- 320M MAU
- 1.1M communities
- 1M posts per day
- 5M comments per day
- 75M votes per day
- 70M searches per day

## Split monolith
- API, search, thing, listing, Rec.

### Listing
- cache after query, invalidate the cache while recieve a new vote ot submit.
  - these can be defered to offline jobs
  - or just mutate in-place within cache, and re-sort with new votes
- traffic jam on vote queue pipeline
  - delayed effect on site noticeable by user
  - add timers for observability
- lock contention
  - patitioning into different queue by topic
- the 99th percentile of jobs perform very poorly
  - cause by request and response time for the website link within posts
  - popular domain may be submitted in many posts
  - split in to subreddit, domain, and profile queues for different type of tasks
- observability is important

### Things
- store objects needed for sorting and filtering in early rdddit in PostgreSQL
- ckustering PostgreSQL via async replication
  - look up data in memcache first, and prefer to use replication for read operation while data not been cached
- while a replica is crash
  1. remove from cluster
  2. found references left around to things that doesn't exist
  3. always starts with a primary saturating its disks, so they decide to upgrade hardware
  4. The reason is the wrong permission setup cause the application can write into secondary db, and cache in the memcache at the same time, but it does not sync to the whole cluster since it is just a secondary db, so query the object written in it on other server will looks like querying an non-exists thing.
- use service discovery system to find database

### Comment Trees
- Use fastlane strategy to isolate the processing of hot threads with massive comment, so they will not slow the whole website down.
  - but the fastline queue still growth dramatically and cause the service goes down
  - set max length for each queue so no queue can consume all of resource

### Autoscaler
- save money off-peak, and automatically react to higher demand
- plan for scale the Zookeeper cluster
  1. launch new cluster in VPC
  2. stop all autoscaler service
  3. repoint autoscaler agents on all servers to new cluster
  4. repoint autoscaler services to new cluster
  5. restart autoscaler services
  6. nobody knows anything happened
- but puppet agent ran and re-enabled the autoscaler services after step 2
  - the services still pointed at old cluster currently
  - meanwhile the new cluster was seen as unhealthy and terminated
  - because all cache server was lost, so the system have to re-warm gentlety or massive read operation will hit PostgreSQL directly
