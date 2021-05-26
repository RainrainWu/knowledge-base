# How Netflix Scales Its API with GraphQL Federation
> [Original Video](https://www.youtube.com/watch?v=QrEOvHdH2Cg)

## Background
- graph are already grown so large, that no single human understand the entire surface area
  - break the API apart, so the domain expert could own their part of graph

## What Federation is?
- Break the API into chnks so that could be developed independently
  - each of them is a micro-aggregator and take care of a single domain

## Three components of a federated architecture
### Graph Services
- GraphQL servers, expose a small portion of overall schema
- publish it using schema registry
### Schema Registry
- to hold the schema of all services and the configurations
### Gateway
- take care single incoming query and break it into subquery which can be executed by the dowstream GraphQL servers.
- break into two step
  - query planning
    - traverse the entire query and identifier the corresponding service of each field
    - determine whether they could be executed parallel or should be sequential
  - query execution

## How does it scale
- already felling pains about monolithic architecture
  - Apollo just release the spec of graphql federation
  - build the gateway base up apollo's reference implementation
- benchmark to test whther it will add too much latency

## It's the future
- it is one of the futures to promote productivity
