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
  4. 
