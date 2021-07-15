# Improving Video Encoding System Efficiency @Netflix
> [Original Video](https://www.youtube.com/watch?v=QEMWEkxMFMM)

## Video Encoding System
- Video is big data
  - Films shot in 4K generate 2~8 TB/day of raw camera footage.
  - VideoSource of 470GB for an hour episode.
  - 1500 types of supported devices

- Pipeline
  - Ingested video -> encoding (parrallel) -> encoded video -> packaging -> packaged video -> deliver to client

## Challenges

- Encoding workloads
  - Bursty and unpredictable
  - Parrallelizable
  - Resource (CPU, RAM, Disk IO) intensive
  - Long period of exception load

### Optimization
- latency
  - How quickly can we light up a title?
  - How quickly can individual work item be completed?
- Throughput
  - How long would it take to re-encode the entire catalog?
  - How much more work can we finish with given resources?

## Three Techniques

### Use high resolution tools
- You can only optimize what you can observe
  - Monitor system metrics
    - idle means you can assign another task during the waiting time
    - single core usage may indicates that the process encountered lock contention
    - Full utilization with multi-core may be the bottleneck of performance, which can give you more rewards for re-writing them

### Leverage hardware features
- Drive your race car with full speed

### Size jobs for better utilization
- More work done with resources
  - Know the shape of different workload, like the blocks of terris.
