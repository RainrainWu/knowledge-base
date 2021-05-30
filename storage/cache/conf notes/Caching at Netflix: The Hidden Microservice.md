# Caching at Netflix: The Hidden Microservice
> [Original Video](https://www.youtube.com/watch?v=Rzdxgx3RC0Q)

## Ephemaral Volatile Cache (EVCache)
- KV store optimized for AWS and tuned for Netflix use cases

### Optimize for AWS
- Instances can be disappear
- Zones fail
- Region become unstable
- Network is lossy
- Customer requests bouse between regions

### Metrics
- Hundreds of terabytes of data
- Trillions of ops / day
- Tens of billions of items stored
- Tens of millions of ops/sec
- Millions of replications / sec

### Architecture
- Offline computation and store the data into cache cluster periodically
  - Fetch each part of cache to compose what the user want
  - Maybe prepare multiple versions of cache for diversity

## Moneta
- Evolution of the EVCache server with SSD.

### Optimization with SSD
- Access patterns are heavily region-oriented
- Hot data is used often, while cold data is almost never touched
  - keep hot data in RAM, cold data in SSD
  - size RAM for working set, SSD for overall dataset
  - 70% reduction in cost
