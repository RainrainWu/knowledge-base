# Redis

- [Redis](#redis)
  - [Features](#features)
  - [Memory Optimization](#memory-optimization)
  - [Persistent Layer](#persistent-layer)
    - [Snapshot](#snapshot)
    - [Operation Log](#operation-log)
  - [Clustering](#clustering)
  - [Cache Warming](#cache-warming)
  - [Eviction & TTL](#eviction--ttl)
  - [Performance Tuning](#performance-tuning)
    - [Benchmarking](#benchmarking)
    - [Slow Log](#slow-log)
    - [Terrible Time Complexity of the Commands](#terrible-time-complexity-of-the-commands)
    - [Big Key](#big-key)
    - [Penetration](#penetration)
    - [Avalanche](#avalanche)
    - [Max Memory](#max-memory)
    - [Fork](#fork)
    - [Turn off Transparent Huge Page](#turn-off-transparent-huge-page)
    - [Switch Append Only File (AOF)](#switch-append-only-file-aof)
    - [CPU Binding](#cpu-binding)
    - [Heavy Swap](#heavy-swap)
    - [Defragmentation](#defragmentation)

## Features
- In-memory key-value pairs storage.
- Rich data type (e.g. array, set, hashmap).
- Single thread (without context) switch for each routine.
  - file event: file read/write.
  - time event: obtain timestamp via system call.

## Memory Optimization
- Redis request memory allocation from OS by arena, which contains many blocks with pre-defined size.
- Different threads will not share the same arena, no resource management sync was needed.
- Rehash the hashmap into a smaller one if it is too sparse.

## Persistent Layer
### Snapshot
  - Written to a file in disk by a forked child process (can be done asynchronous).
  - Rapid recovery for nodes which accidentally down.
### Operation Log
  - Record command to append-only-file before each operation.
  - Flush in-memory records to disk by `fsync` if a certain threshold was exceeded (e.g. #operation, time period)
  - Spread the workload evenly to avoid burst.
  - Accurate point-in-time recovery but takes time to walk through all of required commands.

## Clustering
- Serve request by service port 6379, internal sync by bus port 16379.
- Redis distribute the data into 16384 slots in each database, which is also the maximum master nodes within a cluster (but the recommended number is around 1000).
- Each node will maintain a mapping table between key and node for request routing.
- Stale data issue should be handled if we plan to read data from replicas.
- Force several key to be placed in th same node for accelerating multiple key operation.

## Cache Warming
- Newly added/recovered node hold no data and can lead to low hit rate.
- Recover the data batch by batch (e.g. 1024 slots per 30 sec).
- Enable dual-write to let the new node start to sync with latest data.

**References**
- [Cache warming: Agility for a stateful service](https://netflixtechblog.com/cache-warming-agility-for-a-stateful-service-2d3b1da82642)

## Eviction & TTL
- User can specify TTL (time-to-live) for each key which indicates the expired timing, and the key will never expired if TTL is not set.

- An eviction policy is still necessary to mitigate out-of-space issues, several well known strategies like LRU and random pick are supported.

## Performance Tuning

### Benchmarking
- Launch another redis instance with the same device resource, it can help us clarify whether the performance is limited by resources or it actually act slower (>2x resp time).

- Check max latency within given period.
    ```
    redis-cli -h 127.0.0.1 -p 6379 --intrinsic-latency 60
    ```

- Check the min, max, and avg latency with fixed sampling interval.
    ```
    $ redis-cli -h 127.0.0.1 -p 6379 --latency-history -i 1
    ```

### Slow Log
Redis can help us record the commands take more than given time threshold, as well as the maximum records amount.
```
$ CONFIG SET slowlog-log-slower-than 5000
$ CONFIG SET slowlog-max-len 500
```

### Terrible Time Complexity of the Commands
- Avoid using the commands with complexity greater than O(N) (e.g. SORT, SUNION), which will occupy too much CPU resource.

- Huge N for command with O(N) complexity, which will spend lots of time on network transfer and leads to high network I/O.

### Big Key
- Key with heavy data take much longer time even with simple command (e.g. SET, DEL), the workload on allocation & eviction can also be underestimated.
Scan the big keys.
    ```
    $ redis-cli -h 127.0.0.1 -p 6379 --bigkeys -i 0.01
    ```

- The OPS will raise significantly since the command is the same of global SCAN operation, pay attention to the frequency (can be set with `-i` flag).

- Can enable lazy free to delegate the eviction to background thread, but the synchronization under cluster mode can also be a disaster.
    ```
    lazyfree-lazy-user-del = yes
    ```

### Penetration
- A large amount of request ask for keys not available within redis in a short time, which leads to heavy workload on databases behind.
  - Throttle the traffic against databases behind.
  - Refresh the data within redis asynchronously through background tasks.

### Avalanche
- Redis has two evection policy.
    - Passive eviction Evict only when the key is visited.
    - Active eviction Randomly pick 20 keys and evict the outdated items, the routine will be repeated until the ratio of outdated item less than 25% or takes more than 25ms.

- Vast amount of key evictions in a short time leads to high latency since the active eviction is executed with main thread, which will black the incoming user requests.

- Apply jitter on TTL setting and delegate the eviction to background thread can mitigate the issue, and the expired_key can also ne monitored in advanced.

### Max Memory
- Max memory can be specified through maxmemory configuration, and Redis have to evict several keys before write-related operation if the memory occupation hit the upper bound.

- `allkeys-lru` & `volatile-lru` are common eviction policy, while `allkeys-random` & `volatile-random` brings us better performance but may reduce the hit rate.

### Fork
- Persistent storage & synchronization mechanism will fork from main process, which may share significant resource if the data is high.

- Delegate the persistent storage tasks to slave node and schedule on non-peak hours.

- Increase the value of `repl-backlog-size` to achieve less overall synchronization.

### Turn off Transparent Huge Page
Allocating huge memory page will take more time, we can switch it to never if we tend to make each request be completed faster.
```
$ echo never > /sys/kernel/mm/transparent_hugepage/enabled
```

### Switch Append Only File (AOF)
- `appendfsync always` policy will leads to higher Disk I/O, it is not necessary if we can tolerate a certain extent of data lost.

- `appendfsync no` will be a great choice if we only treat Redis as a cache component without storage requirement.

- Prevent any fsync operation during AOF rewrite/compact.
    ```
    no-appendfsync-on-rewrite yes
    ```

### CPU Binding
- Each type of task within Redis is undertaken by specific process/thread, they can be bound on different CPU core to avoid too much context switch.

### Heavy Swap
- If the memory is not enough for use, disk will be utilized for swap mechanism.


- Fortunately, it can be mitigated directly through add more memory resource.
    ```
    $ cat /proc/$pid/smaps | egrep '^(Swap|Size)'
    ```

### Defragmentation
- Higher fragmentation ratio (the value of mem_fragmentation_ratio within output of INFO command) stands for worse memory utilization and have to be tidied up.

- Defragmentation is also executed by main thread, remember to evaluate the impact on instance before modifying the configurations.