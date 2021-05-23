# Scaling Facebook Live Videos to a Billion Users
> [Original Vedio](https://www.youtube.com/watch?v=IO4teCbHvZw)

# Vedio Platform
- Video upload
- Video clustering
- Distributed video encoding platform
- FB live vedio streamming

## Live Video's differences from Normal Video
- Immediate
- Authetic
- Interative

## History
1. 2015/04 Hackerthon
2. 2015/08 Mentions
3. 2015/12 Users

## Why Live
- Engagement
- Public profiles
- Connecting the world

# Infrastructure
## High Level Archetecture
### Data Flow
Client -> Facebook Edge -> Encoding Service -> Facebook Edge Fan Out -> Playable Cluster
### Resource Usage
|Type|Usage|
|-|-|
|Compute|Decode、Encode、Analysis|
|Memory|Decode、Encode streams|
|Storage|Long term storage of streams|
|Network|Upload and playback of streams|

## Scaling Challenges
### From broadcast client to encoding server
- Number of concurrent unique streams
  - Factors: Ingest protocol, network capacity, server side resource
  - Predictable pattern

### From encoding server to playback clients
- Number of Viewers of all streams
  - Factors: Delivery protocols, network capacity, effective caching
  - Predictable diurnal pattern
- Max number of viewers of single stream, more unpredictable than 
  - Factors: Caching, stream distribution
  - Unpredictable pattern

## Protocols and Codecs
### Solution v.s. Requirements
||Time to production|Network compatinility|E2E latency|Application size|
|-|-|-|-|-|
|WebRTC||X|||
|HTTP|||X||
|Custom Protocol|X||||
|Proprietary||||X|
|RTMP(S)|V|V|V|V|

## Stream Ingestion and Processing
- CLient first upload their video to facebook edge, than the encoding server within data centers ingest from them.

## Reliability Challenges
### Network
- Use adaptive bit rate based on connectivity
- Handling temporary connectivity loss on broadcasts
- Audio only broadcasts and playback

### Thundering Herd
- Find the sweet spot of cache timeout.

## Lesson Learned
- Large services can grow from small beginings.
- Adapt service to the network environment.
- Reliability and scalability are built into the design, not added later.
- Hotspots and thundering herds happen in multiple components.
- Design for planned and unplanned outage.
- Make compomises in order to ship large projects.
- Keep the architecture flexible for future features.
