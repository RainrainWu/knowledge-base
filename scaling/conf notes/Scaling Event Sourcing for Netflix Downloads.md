# Scaling Event Sourcing for Netflix Downloads
> [Original Video](https://www.youtube.com/watch?v=rsSld8NycCU)

# Why need a Downloads License Accounting Service
- Wfile press the play button
  1. get context with metadata
  2. get license for decrypt and encrypt content
  3. session events send to server(start, pause, resume, keepalive)
- While press the stop button
  1. release license
  2. session event stop 

## Rquirements
- Flexible: data model can be changed
  - use document model with versioning but may not be debuggable
- Debuggable: event sourcing
- Reliable: Fallbacks
- Scalable 

# Event Sourcing Overview
Rest Endpoint -> Aggregate Service -> Aggregate Repository -> Event store

# Deep Dive into the Event Sourcing Architecture

# Working with the Livense Accounting Service
