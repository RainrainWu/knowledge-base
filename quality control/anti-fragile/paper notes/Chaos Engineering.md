# Chaos Engineering

> [Original Paper](https://arxiv.org/pdf/1702.05843.pdf)


## Abstract
- Distributed System has various failure modes.
  - Use experimentation to verify the reliablity.
  - Chaos Engineering team to discussing underlying principles and how to use it.

## Introduction
### Chaos Monkey Service
- Randomly selects virtual machine instances that host our production services and terminates them.
- encourage Netflix engineers to design software services that can withstand failures of individual instances.
- Only Active during normal working hours.
- "Chaos Kong" exercises that simulate the failure of an entire Amazon EC2 region.
- Common themes underlying these activities more subtle than simply "break things in production".
- the discipline of experimenting on a distributed system in order to build confidence in its capability to withstand turbulent conditions in production.

## Netflix Architecture
### Infrastructure
#### Open Connect
- CDN Network
- Mainly serve the vedio streamming

#### Control Plane
- Implements all the functionality in the UI other than streaming the the video itself.
- replicated across three geographic regions to reduce latency and improve availability. 

## Shifting to a system perspective
- view a collection of services running in production as a single system.
  - better understand the behavior of this system by injecting real-world inputs
    - transient network failures
    - surges in incoming requests
    - malformed data inputs
  - observing what happens at the system boundary.
- when describing distributed systems, functional specifications are incomplete; they
do not fully characterize the behavior of the system for all possible inputs.
  - difficult to fully enumerate all inputs
  - impractical to compose specifications of the constituent services in order to reason about behavior.
- ask questions like "does everything seem to be working properly?" and "are users complaining?" rather than "does the implementation match the specification?"

## Principles of Chaos Engineering
- Chaos Engineering revolves around running experiments.
- Design an experiment requires specifying
  - hypotheses
  - independent variables
  - dependent variables
  - context
- principles which embody the Chaos Engineering approach to designing experiments
  - Build a hypothesis around steady state behavior
  - Vary real-world events
  - Run experiments in production
  - Automate experiments to run continuously

### Build a hypothesis around steady state behavior
### Vary real‐world events
### Run experiments in production
### Automate experiments to run continuously
