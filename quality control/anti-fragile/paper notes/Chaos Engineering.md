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
- "works properly" is too vague as a basis for designing experiments.
  - the quality attribute we focus on most is availability
  - even if one of the internal services fails, this does not necessarily have an impact on overall system availability.
- use fallbacks to ensure graceful degradation: that failures in non­critical services
have a minimal impact on the user experience.
  - if there is a failure in the caching service, then service A can fall back to making a request directly against service B.
  - provide personalized content, where the fallback would be to present a reasonable default.
- Ultimately, what we care about is: "Are users able to find some content to watch, and
successfully watch that content?"
  - operationalize this concept by observing how many users start streaming a video each second.
  - metric SPS, for (stream) starts per second.

#### Metrics SPS
-  primary indicator of the overall health of the system
-  fundamental assumptions is that complex systems exhibit behaviors that are regular enough that they can be predicted.
  - SPS metric varies slowly and predictably over the course of a day
  - If engineers observe an unexpected change in SPS, we know there is a problem with the system.
- a good example of a metric that characterizes the steady-state behavior of the system
- a strong, obvious link between these metrics and the availability of the system
  - hypothesize that failing over from one region to another will have minimal impact on SPS.
  - not focus on finer grain metrics such as CPU load ot query time 
  - steady state characterizations that are visible at the boundary of the system, which directly capture an interaction between the users and the system.
  - conclude an experiment early if fine-grained metrics indicate that the system is not functioning correctly, even though SPS has not been impacted.

### Vary real‐world events
- corner cases and error conditions will happen out in the real world, even if they don't happen in our test cases
  - Clients send malformed requests
  - third party services we consume send malformed responses
  - hard disks fill up
  - memory is exhausted
- 92% of catastrophic system failures were the result of incorrect handling of non­fatal errors
  - look at historical data to see what type of inputs were involved in previous system outages
  - any input which you think could potentially disrupt the steady­state behavior of the system is a good candidate for an experiment.
    - terminate virtual machine instances
    - inject latency into requests between services
    - fail requests between services
    - fail an internal service
    - make an entire Amazon region unavailable
    - the rate of requests received by a service
- may need to simulate the event instead of inject it
- use their judgment to make trade-offs between the realism of the events and the potential risk of harm to the system.
  - production financial trading system
- selectively apply the event to a subset of users
  - make an internal Netflix service behave as if it is unavailable from the point of view of some users, but for others the service appears to function properly

### Run experiments in production
- traditional software testing approaches are not sufficient for identifying the potential failures of distributed systems
  - A server is overloaded, and takes an increasing amount of time to respond to requests. One of the clients places outbound requests on an unbounded local queue. Over time, the queue consumes more and more memory, causing the client to fail.
  - A client makes a request to a service that is fronted by a cache. The service returns a transient error which is incorrectly cached. When other clients make the same request, they are served an error response from the cache
- not possible to fully reproduce the entire architecture and run an E2E test.

### Automate experiments to run continuously
- System changes continuously over time.
  - confidence in the results of past experiments decreases over time.
- the experiments will ultimately need to be automated in order to ensure that they run over and over again as the system evolves

## Running a Chaos experiment
- Based on the principles, define an experiment:
  - Start by defining ‘steady state’ as some measurable output of a system that indicates normal behavior.
  - Hypothesize that this steady state will continue in both the control group and the experimental group.
  - Introduce variables that reflect real world events like servers that crash, hard drives that malfunction, network connections that are severed.
  - Try to disprove the hypothesis by looking for a difference in steady state between the control group and the experimental group.

## The Future of Chaos Engineering
- the increasing complexity of software systems will continue to drive the need for empirical approaches to achieving system availability.

### Move Forward
- Case studies from other domains
  - hope that more organizations will document how they are applying Chaos Engineering to demonstrate that these techniques are not unique to Netflix.
- Adoption
  - What approaches are successful for getting an organization to buy into this approach, and for getting engineering teams within the organization to adopt it?
- Tooling
  - build a set of tools for running these types of experiments that are reusable across organizations
- Event injection models
  - Research shows that many failures are triggered by combinations of events rather than single events
  - How do we decide what set of experiments to run?
