# Scaling
- 隨著服務的使用者越來越多，儲存的資料量級越來越大，原有的 PostgreSQL server 可能逐漸不堪負荷而需要考量擴展的問題。

## Vertical Scaling
- 最直接的做法就是透過為機器增加更多資源，比如 CPU、RAM 等等。
- 但這可能僅是治標不治本的方法，仍有受限於 connection 數量限制、沒有容錯性或被冗余資料拖慢的問題。

## Horizontal Scaling
- 透過增加更多 PostgreSQL instacnes 進行流量負載平衡，但由於資料需要強一致性所以設定 instance 之間的溝通也較為麻煩。
- 由於個別的 instance 之間彼此相互獨立，有自己的 connection quota，也能在其中一台 instance 意外下線時接手處理流量。

### pgpool-II
- pgpool-II 是 PostgreSQL 的一個熱門擴充工具，他的角色類似於擔任代理 PostgreSQL cluster 接收 application server 流量的 proxy。
- 其主要功能包含：
  - Load balancing
    - 將讀取資料的 query 在多個 read-only PostgreSQL 上做負載平衡以增進效能。
  - Connection pooling
    - 負責統一管理對 PostgreSQL 的 connection 並重複使用省掉不斷建立 connection 的開銷。
  - Replication
    - 管理資料在多個 PostgreSQL instance 之間的複製流程，以避免單一 instance 損毀就造成資料缺失的風險。
    - 雖然其支援許多不同的複製流程，但官方仍是推薦使用 PostgreSQL 自身的 Write-ahead Log 來實現的 streamming replication。
    - [What is streaming replication, and how can I set it up?](https://www.postgresql.fastware.com/postgresql-insider-ha-str-rep)
  - Automatic failover
    - 邏輯上僅有一個 PostgreSQL instance 作為 primary instance 來接收 write 的操作，當他意外下線時可自動轉移 primary 資格讓其他 instance 遞補。
    - 透過定期向各個 PostgreSQL instance 發送 heartbeat，偵測所有節點可用性並在節點失效時自動觸發 failover 機制。
  - Online recovery
    - 當有任何 PostgreSQL instance 意外下線需要 recover 時，或是需要動態擴充 cluster 大小時都無需停止運作。
  - Watchdog
    - 除了讓多個 PostgreSQL instance 形成 cluster 實現 HA 外，pgpool-II 本身也可以組成 cluster 以實現 HA 的目的。
    - 透過 watchdog 機制連結多個 pgpool-II 相互溝通，若目前的 active node 下線時便可以執行 failover，而對外仍是以同一個 virtual IP 溝通。
- [PostgreSQL High Availability using pgpool-II](https://www.postgresql.fastware.com/postgresql-insider-ha-pgpool-ii)

#### 潛在問題
- pgpool-II 固然是繼承了 cluster 的優勢，但反過來說也繼承了對應的風險：
  - Split brain
    - 若 cluster 中各節點有 master-slave 的階級差異時，往往只能有一個 master 的關鍵節點，存在兩個以上時就可能導致邏輯不一致的問題。
    - 當 failover 機制意外觸發或執行不完全就可能發生，比如重大網路因素導致 heartbeat 慢了幾秒回覆，但原本的 primary instance 仍在運作卻觸發了 failover。
    - 當 pgpool-II 的 active node 無法連接當前 PostgreSQL cluster 的 primary instance 時，會徵求其餘的 pgpool-II 參與投票，以多數決來判斷是否該 PostgreSQL instance 真的掉線需要進行 failover，否則就轉移自身 active node 的資格給能夠連接 primary instance 的 pgpool-II 節點。
    - 為了使投票結果更可靠，pgpool-II 的節點數建議為三個以上的奇數。
