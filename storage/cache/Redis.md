# Redis
- Redis 是一個以 key-value pair 為儲存結構，支援的資料型別也包含 array, set, hash 等等多元資料型別。
- 將資料保存在記憶體中以提昇操作效率。
- 雖然記憶體中的資料是揮發性的，但 Redis 也可以支援定時 snapshot 寫入持久化儲存的備份功能。

## Event Loop
- Redis 會用 thread pool 管理多個 thread 個別專職負責不同的事務，比如叢集同步、GC、讀取寫入操作等等。
- 但本質上各個功能都是 single thread 的，主要功能的 event loop 也是，這使得他不會有 context switch 的消耗。
- Event Loop 主體是個多工器的 select 機制，會依照當前的任務、作業系統、資源用量來決定操作方式。
- 其負責的任務可分為兩大類，分別是 file event 和 time event。
  - file event 多半是檔案的讀寫，會有個 timeout 的期限，預設是下一個 time event 的時間點。
  - time event 多半是 redis 的例行作業，比如 `serverCron` 會透過 system call 取得全局 timestamp 並和其他任務共享節省消能開銷。

## Memory Optimization
- Redis 在向 OS 請求 allocate 記憶體資源時是以 arena 為單位的，內部已經有大量預先切分好的不同大小的記憶體區塊。
- 預先切分好的記憶體大小好處是不用每次請求時花時間計算大小切分，也不會因為大小不一的記憶體區塊即使回收後仍造成記憶體空間破碎的問題。
- 負責不同任務的不同 thread 彼此之間不會共用 arena，因此不用同步資源控管的問題。
- Redis 同時也有對於 hash table 稀疏程度容忍的臨界值，一旦儲存過於鬆散就會啟用另個較小的 hash table 及非同步 rehashing 機制避免佔用過多閒置資源。

### References
- [jemalloc](https://github.com/jemalloc/jemalloc)
- [In-Memory Computing At Aerospike Scale: When To Choose And How To Effectively Use JEMalloc](http://highscalability.com/blog/2015/3/17/in-memory-computing-at-aerospike-scale-when-to-choose-and-ho.html)

## Persistence Layer
- Redis 雖是在 RAM 層面為主的應用，但也能支援持久化的功能，大致分為 snapshot 和 operation log 兩種概念實作。
- snapshot 的做法是透過 Redis Database 定期將當前資料寫入 disk，會透過 fork 出來的一支 child process 負責。
  - 在 snapshot 的過程會啟用 copy-on-write 的機制同時持久化新的寫入，因此不需要 block 著正在使用中的 table。
  - 雖然 child process 會取走 parent process 的部分資源造成效能降低，但是可以透過時間間隔或是操作數間隔來選定合適的時間錯開 rush hours。
  - 持久化下來的 snapshot 也能隨取即用，適合用來 recover 意外死掉的節點。
- operation log 的做法是透過 append-only files (AoF)，會在每次寫入操作的同時寫紀錄下操作內容。
  - 多數時候操作記錄會先記錄在 RAM 中，等待一段時間或是一定操作次數後才透過 `fsync` 寫入 disk。
  - 歷史性的紀錄可以將效能損耗較為均分到時間線上，也能提供 point-in-time recovery，但就是要一道道指令執行沒辦法馬上使用。

## Clustering
- 可以透過建立多個 redis 節點來組成 cluster，除了能分散儲存資料外，也能受益於 master-slave 備份機制對抗節點失效的風險。
- 組成 cluster 中的每個 redis 節點都必須開啟兩個 port
  - 一個對外接收 request 的 data port 預設是 6379
  - 一個是節點之間互相溝通 (e.g. 更新設定值、確認失效節點) 的 cluster bus port，固定比前者 +10000，預設是 16379
- redis 是透過把資料群分在總計 16384 個 slot 中儲存的，背後的 hash slot 演算法可幫助他盡可能地平均分散資料並降低遷移時的運算成本。
  - 有多個節點時各個節點會負責管理一部份的 slot（至少一個），最大的叢集可有 16384 個節點，但官方建議最大值是 1000 個。
  - 每個節點各自會維護一份 slot 與其儲存節點的 mapping 資訊，當收到請求中的 key 不屬於自身管理的 slot 時，會 redirect 至目標節點處理。
  - 一般情況下一個 slot 的資料讀寫只由一個 master 來處理，而其他 slave 主要用來備份，若因高流量考量分彈讀取操作的話需考慮更新延遲的資料一致性問題。
  - 同時可以透過 hash tag 的語法 `{TAG_NAME}KEY_NAME` 進一步強行把同個 hash tag 的 key-value pair 放在同個 slot 中，以加速 multiple key 的操作。

### 新增節點流程
> 由於 redis 儲存的資料是由各個 slot 分開獨立管理的，可自由搬動其所在節點，因此可隨時任意新增節點不影響運作。
1. 啟動一個新的 redis 節點
2. 透過 cluster add-node 的指令把新的節點加入現有的 cluster。
3. 確認新節點已加入後，透過 cluster reshard 指令遷移部份 slot 到新的節點中，或者用 cluster rebalance 指令以權重分配 slot 數量。

### 版本升級流程
> 核心概念是用 rolling upgrade 以保證整個 cluster 對外的服務不中斷。
1. 啟動和當前 master 數量相同的新版本 redis 節點，並加入 cluster 形成一對一的 slave 以複製資料。
2. 資料複製完成後便可開始手動進行 failover 將 master 角色轉移至新版本，轉移時可能會有導致系統不穩風險，是系統對穩定度的要求和不統版本間組成 cluster 的相容性來決定要一次性全部 failover 還是逐個完成
3. 新版本的 master 轉移完成後，逐步啟用新版本的節點作為 slave 加入 cluster，並逐步汰換舊版本的節點至全數更新為止。

### Cache Warming 流程
1. 在擴展 cluster 或是有節點臨時失效需要重啟時，為了避免新啟動的節點因不含任何資料而形同虛設，導致仍有 hit rate 驟降且大量請求湧入資料庫的問題，透過 warming 機制來讓新節點快速引入當前關鍵資料。
2. 但同時需要考慮若直接從當前運作中的節點進行大規模的 dump 的話，可能會影響 client 端的使用狀況。
3. 比較折衷的做法是先把所有的 key 透過 slot 的資料 dump 下來後，切分成不同 chunk，再由一些 worker node 分工去 get 到所有 key 的 value 填入新的節點中。
  - 由於第一步只抓取 key 的操作只需要 slot 的 metadata 即可不需要真的存取所有的 value，因此效能比較不會有劇烈影響。
  - 讓多個 worker node 像 client 一樣去取 value 的好處是和 client 一樣從外部存取可以直接被負載平衡避免只針對單節點，同時也可以依據時間需求或當前工作量調整 worker 數量。
4. 多數時候也可以在新增節點時就設定 dual-write 機制讓最新的 cache 同時也寫入到新節點中，而從其他節點複製來的資料因為會有時間差延遲所以就要避免覆蓋到新資料。

- [Cache warming: Agility for a stateful service](https://netflixtechblog.com/cache-warming-agility-for-a-stateful-service-2d3b1da82642)

## 常見問題

### 空間限制
- 資料庫所儲存的資料量肯定會比 cache 儲存的資料量大的多的，而若要受惠於 cache 的機制通常會先向資料庫取得資料後存入 cache 再回覆給使用者，久而久之 cache 的空間便會有耗盡的問題。
- Redis 針對這問題有兩個方法可以解決一個是設置每對 key-value pair 的過期時間，也就是 TTL (time-to-live)，另個是設定強制淘汰機制 eviction policy。

#### TTL (time-to-live)
- Redis 中可以設置每對 key-value pair 的過期時間，若沒特別指定的話預設就是無限期保存。
- 但為了效能和可用性考量所以不會週期性的確認每一個 key-value pair 的效期，而是隨機抽樣其中幾個，若有發現效期已經過的就會移除。
- 同時在使用者嘗試存取一對 key-value pair 時，也會先檢查其效期是否過期，以避免娶到過期的數值。
- 但以上兩種方法並無法完全避免儲存空間用盡的狀況，因此仍需要強制淘汰機制。

#### Eviction Policy
- 當所剩空間不足以存下新資料時，就需要選擇一些 key-value pair 來強行刪除以騰出足夠空間，而如何選擇就是透過這個 policy 來實現。
- 目前提供針對未設置 TTL 期限之資料的 random, lru，和有設置 TTL 期限之資料的 random, lru 等等策略，但若選擇針對有設置 TTL 的資料淘汰卻沒有為任何一筆資料設定 TTL 的話就會引發錯誤。

### Cache Penetration
- 具體情況就是瞬間湧入大量的請求取存取 cache 中沒有留存的資料，導致所有請求都直接以資料庫為對象，cache server 形同虛設。
- 這種情況多半是由惡意攻擊引起，解決方法有許多彈性，比如：
  - 維護一定數量的 lock 來限制同時間向資料庫存取的壓力，只有取得 lock 的才能主動向資料庫拿資料。
  - 把向資料庫存取資料的任務改為非同步執行，可針對請求直接返回 server busy，但背景中的 async job 仍會更新 cache 中的值。
  - 透過一些 middleware 先行擋住疑似腳本發起的請求。

### Cache Avalanche
- 具體情況是短時間有大量的資料到期，導致大量的請求也會直接以資料庫為存取對象，cache server 形同虛設。
- 此類問題類似於分散式系統的 thundering herds，只需要在設定 TTL 時乘上一個 random number 上每筆紀錄的到期時間有所差異。

### 併發寫入
- 當多個 task 併發執行並針對同一筆紀錄做更新時，可能爭搶 write lock。
- 若是單一 redis server 的情況下可以用 transaction 強行持有執行權限直到所有 command 依序執行完成，期間不會受到其他操作干擾，也就不會有搶 lock 的問題。
- 但在多節點的 cluster 環境中 transaction 並無法規範分布在不同節點上的資料，因此有點雞肋不一定能達成效果。
- 若不要求執行順序的話，就在 application tier 設計一個鎖來控制，否則就是戴上時間戳後放入 queue 來依序執行。

## Performance

### Trouble Shooting While Slow Down

#### Benchmarking
- Launch another redis instance with the same device resource, it can help us clarify whether the performance is limited by resources or it actually act slower (>2x resp time).
- Check max latency within given period.
  ```bash
  redis-cli -h 127.0.0.1 -p 6379 --intrinsic-latency 60
  ```
- Check the min, max, and avg latency with fixed sampling interval.
  ```bash
  $ redis-cli -h 127.0.0.1 -p 6379 --latency-history -i 1
  ```

#### Slow Log
- Redis can help us record the commands take more than given time threshold, as well as the maximum records amount.
  ```bash
  $ CONFIG SET slowlog-log-slower-than 5000
  $ CONFIG SET slowlog-max-len 500
  ```

#### Terrible Time Complexity of the Commands
- Avoid using the commands with complexity greater than O(N) (e.g. SORT, SUNION), which will occupy too much CPU resource.
- Huge N for command with O(N) complexity, which will spend lots of time on network transfer and leads to high network I/O.

#### Big Key
- Key with heavy data take much longer time even with simple command (e.g. SET, DEL), the workload on allocation & eviction can also be underestimated.
- Scan the big keys.
  ```bash
  $ redis-cli -h 127.0.0.1 -p 6379 --bigkeys -i 0.01
  ```
  The OPS will raise significantly since the command is the same of global `SCAN` operation, pay attention to the frequency (can be set with `-i` flag).
- Can enable lazy free to delegate the eviction to background thread, but the synchronization under cluster mode can also be a disaster.
  ```bash
  lazyfree-lazy-user-del = yes
  ```

#### Avalanche
- Redis has two evection policy.
  - **Passive eviction**
    Evict only when the key is visited.
  - **Active eviction**
    Randomly pick 20 keys and evict the outdated items, the routine will be repeated until the ratio of outdated item less than 25% or takes more than 25ms.
- Vast amount of key evictions in a short time leads to high latency since the active eviction is executed with main thread, which will black the incoming user requests.
- Apply jitter on TTL setting and delegate the eviction to background thread can mitigate the issue, and the `expired_key` can also ne monitored in advanced.

#### Max Memory
- Max memory can be specified through `maxmemory` configuration, and Redis have to evict several keys before write-related operation if the memory occupation hit the upper bound.
- `allkeys-lru` & `volatile-lru` are common eviction policy, while `allkeys-random` & ``volatile-random` brings us better performance but may reduce the hit rate.

#### Fork
- Persistent storage & synchronization mechanism will fork from main process, which may share significant resource if the data is high.
- Delegate the persistent storage tasks to slave node and schedule on non-peak hours
- Increase the value of `repl-backlog-size` to achieve less overall synchronization.

#### Turn off Transparent Huge Page
- Allocating huge memory page will take more time, we can switch it to `never` if we tend to make each request be completed faster.
  ```bash
  $ echo never > /sys/kernel/mm/transparent_hugepage/enabled
  ```

#### Switch Append Only File (AOF)
- `appendfsync always` policy will leads to higher Disk I/O, it is not necessary if we can tolerate a certain extent of data lost.
- `appendfsync no` will be a great choice if we only treat Redis as a cache component without storage requirement.
- Prevent any fsync operation during AOF rewrite/compact.
  ```
  no-appendfsync-on-rewrite yes
  ```

#### CPU Binding
- Each type of task within Redis is undertaken by specific process/thread, they can be bound on different CPU core to avoid too much context switch.

#### Heavy Swap
- If the memory is not enough for use, disk will be utilized for swap mechanism. Fortunately, it can be mitigated directly through add more memory resource.
  ```bash
  $ cat /proc/$pid/smaps | egrep '^(Swap|Size)'
  ```

#### Defragmentation
- Higher fragmentation ratio (the value of `mem_fragmentation_ratio` within output of `INFO` command) stands for worse memory utilization and have to be tidied up.
- Defragmentation is also executed by main thread, remember to evaluate the impact on instance before modifying the configurations.
