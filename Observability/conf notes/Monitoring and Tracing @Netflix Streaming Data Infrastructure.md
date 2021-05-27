# Monitoring and Tracing @Netflix Streaming Data Infrastructure
###### tags: `Netflix`, `Kafka`, `Flink`
> [Original Video](https://www.youtube.com/watch?v=DlWYNoLmma8)

# Vision
## Four levels (from low to high)
1. Observability:
  - to have clear insight to the product
2. Availability
  - data keeps flowing end to end without interruption
3. Operability
  - most of the operations should be automated
4. Data Quality
  - Key indicators for data quality
  - lost rate, duplicate rate, end to end latency

### Observability
- Kafka monitoring
  - determine whther it is operational by heart beating
  - metadata: broker state, ISR stae, controller liveness
  - consumer lag = log end offsets - commit offsets
  - internal metrics become stale when consumer process stops

### Abailability & Operability
- AUto remediation
  - automatically relanuch for staeless jobs on first stuck allert
    - calculation for enough autoscaling capacity to catch up well while traffic growth dramatically during peak hours

### Data Quality
- loss detection
  - data still may lost even if send count equals to recieve count
    - because of duplicattion
    - identification information was lost in aggregation
- auto recovery
  - trace data as they move in the pipeline
    - but data lineage and dependency graph is not included since it can be found on control plane
    - focus on figuring out who doesn't make that call
    - launch another Kafaka cluster and record all of checkpoints information through the whole data pipeline and group them by trace id
  - major categories of data loss
    - nature of less durable configration
    - extreme situations
    - human/code error
