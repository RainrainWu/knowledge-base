# Solving Mysteries Faster with Observability

## Observability

### tooling
- Logs : detailed view into individual service
- Metrics : how services are performing in a macro-scale
- Traces : records individual request through all system

### content
- metadata is key to understand the asset, device type, country, account preference, etc.
- how do we make understanding the system's behaviour fast and adjustable?

## Request Tracing
- tracking the activity resulting from a request to an application
  - trace the path of a request as it travels across a complex system
  - discover the latency of the componebts along that path
  - know which componebt in the path is creating a bottleneck
- graph a directed call-graph and a chronological waterfall timeline
- adopt edgar
  - playback session include logs and metadata
  - capture 100% of playback related data so that edgar's users can rely on it having data about an issue
  - should covers a wide spread of services
  - serves multiple user groups, from engineers to customer support
- use traceID in logs
- test out 100% tracing of some business-critical subset
