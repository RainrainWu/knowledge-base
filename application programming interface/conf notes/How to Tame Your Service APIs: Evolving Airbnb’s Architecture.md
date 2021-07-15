# How to Tame Your Service APIs: Evolving Airbnb’s Architecture
> [Original Video](https://www.youtube.com/watch?v=yI91FSghAL4)

## Context
- At the beginnig of Airbnb, the arichetecture is a simple RoR monolithic service, but it becomes complicated overtime.
  - Decide to migrate to service-oriented architecture (SOA).

## Scaling Challenges

### Context
- The number of service growth dramatically.
  - 50 services at 2016, handle by 500+ engineers
  - 550+ services at 2020, handle by 2000+ engineers

### Challenges of SOA v1
- Complcated dependency graph.
- Slower developer velocity due to much more integration touch points.
- Fractured data domain and business logic.
- Repeated patterns or similar code in different services.

## Design Principles
- Simplify
  - The dependency graph
  - The developer experience
  - The access control

### SOA v2
- Abstraction
  - Internal services
  - Data aggregators
  - Service blocks
- GraphQL for aggregators


## Abstraction APIs

## Operating APIs
