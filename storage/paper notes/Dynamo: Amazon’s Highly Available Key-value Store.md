# Dynamo: Amazon’s Highly Available Key-value Store
> [Origin Paper Link](https://dl.acm.org/doi/abs/10.1145/1323293.1294281?casa_token=Vhd9CIMdUUEAAAAA%3Aiz1gZeBIcCLgkzCxdoWZC4G2VWkMQdx62srpHJo18ZsT2o6RqBy-6MGsUEi8XGZ2LhqdRWGRBnS7)

## Context
- 在上萬節點規模的叢集上追求可靠性是非常有挑戰性的，因為節點失效和網路問題會持續的發生，並影響使用者的體驗。
  - 最理想的解決辦法不是想辦法避免他，而是直面著個問題並設法與他共存。

## Introduction
- 其中一個學到的領悟是一個系統的 reliability 和 scalibility 很大程度的依賴於如何儲存與管理 state 相關的資料。
  - 比如遇到網路波動、硬碟壞軌、或是洪水沖毀某個 data center 時都仍要讓使用者能放商品到購物車中。
- Amazon 的許多服務只需要透過 primary key 來獲取資料，比如商品訊息、賣家資訊等等。
  - 不過使用傳統的 RDB 會在 performance 和 scaling 上造成許多局限，因此 Dynamo 期望能提供一個同樣是基於 primary key 來取的資料的全新解決方案。
- Dynamo 應用了許多常見提升 availibity 的方法，比如透過 consistent hashing 實現 partition 和 replication。
- 而 consistency 的需求則有使用到了 versioning 和 quorum-like 的多數共識門檻，以及 gossip based 的節點失效偵測機制。
- 簡單而言，Dynamo 實現了一個幾乎不需要手動維護，加增或刪除節點時也不須重新部署的 HA KV store。

## Background
- 許多 Amazon 的服務是使用傳統的 RDB 進行資料儲存，不過往往只需要透過 primary key 來取得資料。
  - RDB 所提供的進階管理功能和複雜的 query 並不必要，反而造成無謂的效能損耗。
  - 另外多數的 RDB 往往把 consistency 放在 availability 前面。

### System Assumptions and Requirements
- 適用於 Dynamo 的服務需具備以下特點：
  - query model 的讀寫操作都可以透過一個 primary key 定位到目標紀錄，也不需要使用複雜的 relational schema。
  - 較弱的 ACID 需求，根據過去的經驗過高的 ACID 要求會導致效能上的瓶頸，而 Dynamo 適度的降低了 consistency 的等級。

### Design Considerations
- 當我們為了 availability 而在 consistency 上稍作讓步後，我們便需要處理何時該解決 conflict 以及由誰解決的問題。
  - 傳統做法上多數時候因爲讀取操作遠多於寫入操作，因此會集中在寫入操作時才 resolve conflict 以確保大量讀取操作能被快速完成。
  - 但 Amazon 的許多操作如加入購物車和更改店家資訊若因為一致性問題而被暫時拒絕寫入操作，就會很大程度影響使用者體驗。
  - 所以反其道而行只在 read 時處理 conflict 從而保證寫入操做絕對可行。
- 整個 Dynamo 的設計大至尊從以下幾個原則
  - Incremental scalability：能夠一次擴展一台節點並最小化對使用者和系統的影響。
  - Symmetry：每個節點的職責應該和他的 peers 完全相同，不應該存在嘟一無二的節點。
  - Decentralization：前者的強化，捨棄中央控制轉為追求去中心化，此舉能大幅增加可擴展性和可用性。
  - Heterogeneity：能夠接納系統中存在不同規格的節點，並依據資源調度最合適的工作量。

## RELATED WORK
### Peer to Peer Systems
- P2P 系統的幾代演進
  - 初代的 P2P 系統主要用於檔案分享，比如 Freenet 和 Gnutella，這時期的連線通常是隨機建立的，處理 query 時會大量 Broadcast 至其他節點並盡可能獲得更多資訊。
  - 接著演化出了 Structure P2P 的架構，透過統一的 protocol 來讓任意節點可以快速的把 query 導向擁有目標資料的其他節點並有效降低 hop 數，比如 Pastry 和 Chord。
    - 如果讓節點中都存有足夠的 routing 資訊的話，也是有可能降至 O(1) 的時間複雜度的，意味著立即 route 至確定有目標資料的節點。
  - 更後期的 Oceanstore 和 PAST 正式建立於這些基礎上，並加上了 concurrency control 和 resolve conflict 的功能。

### Distributed File Systems and Databases
- 在許多 P2P 系統中僅能使用 flat namespaces，這點在較近代的分散式檔案系統中所提供的 hierarchical namesapce 獲得解決，比如 Ficus 和 Coda。
- 而 resolve conflict 的部分也有很多現行的解決方案，比如 google file system，只是它使用了 master node 的位階制度。
- 另外也有像 Bayou 這類對於斷線後獨立運作的能力的實現，在應對 network partition 上有所表現也大大增加了 partial tolerence 的特性。
- 和上述幾個實踐相比，Dynamo 的 key value store 有一些較為明顯的優勢。
  - 儲存的單元物件位變得更小 (< 1MB)。
  - 更方便的對各個 application 的資料分開做設定，應該是不同 key-value 之間切的很乾淨的關係。

### Discussion
- 到這裡可以列出 Dynamo 的主要需求。
  - 首要目標就是 always writeable。
  - 預設信任所有節點，可以自主運作並回傳資料。
  - 不需要階層式 namespace 或複雜的 schema。
  - 99.9% 的讀寫操作都必須在數百 ms 內完成。
- 其中分散式系統中處理 query 時 multi-hop 的行為是拖慢響應速度的主因，因此 Dynamo 的每個節點都維護足夠的 routing 資訊直接導向儲存目標資料的節點，進而把時間複雜度壓在 O(1)。

## SYSTEM ARCHITECTURE
### System Interface
- 由於是 key value 的儲存，因此 Dynamo 主要提供兩個 API，get 和 put，同時附帶 context 物件提供版本等等資訊。

### Partitioning Algorithm
- 為了能接受動態的增加節點，Dynamo 透過 consistent hashing 進行 partition 並散佈到系統中，也正是常見的 hash ring。
- hash ring 的分佈邏輯讓叢集新增或刪除節點是，可以只調整其在圓環上相鄰的 partition 即可，其他的部分不會被影響。
- 不過基礎的 consistent hashing 也有一些顯而易見的問題，比如不均勻分布，以及無法適應叢集中多種不同規格節點的異質性。
- 為了應對此問題，Dynamo 會再將 physical node 切分為規格更加一致的 virtual node，並散佈在 hash ring 的不同位置上，而 virtual node 的好處包含
  - 當街點失效時，其所負責的多個 virtual node 的工作量能夠被分攤到更多機器上。
  - 當心節點加入時，給予合適數量 virtual node 可以讓他獲得和其他節點大致相同的工作量。
  - 可以基於機器資源的不同給予不同數量的 virtual node。

### Replication
- 備份的部分也是利用了 hash ring 的結構優勢，透過一個 replica factor 的參數 k，在環狀位置上的前 k 個片段所負責的節點做備份。
  - 比如當前節點是 N，而 replica factor 是 2，那就會在 N - 1 和 N - 2 兩個節點上備份 N 的資料。

### Data Versioning
- 由於 Dynamo 提供的事 eventually consistency，所以寫入操作是容許非童多傳遞至其他節點的，但也因此需要 version 的概念確定先後順序。
  - 其中有有許多很棘手的情況，比如說兩筆新加入購物車的商品被寫入到了不同節點，那這輛個節點應該如何融合？
    - 單純用 last writer win 操略的話，可能其中一個商品就會被清除掉，但這是不可以的所以要想辦法避免。
  - 一般常見的 inconsistency 可以透過釐清 causality 來解決，Dynamo 主要是靠 vector clock。
  - 當出現版本紛歧的時候，需要一些特別的 reconciliation 來解決衝突，細節的部分類似 CRDT 的概念，難以透過筆記解釋建議直接查。
- 對於每個 update 操作，Dyanmo 會要求使用者必須在 context 內指名期要徵新的物件版本。

## Execution of get () and put () operations
- 基於信任所有節點的前提，所有 Dynamo 的集點都可以說 get 和 put 操作並回傳對應資料。
- 而一個 request 會被導向哪個節點處理基本上有兩個決策管道。
  - 一個是透過 generic load balancer 透過 load information 作分流。
  - 另個是透過能夠存取所有節點資訊的 client lib 來直接導向負責目標資料的節點。
  - 前者可以和 Dynamo 解耦，後者則是能有效迴避潛在的 forward latency。
- 接收了 request 並承擔 return 責任的節點被稱為 coordinator，理論上通常是針對該 key 操作的理想節點清單中的第一名。
- 每個 key 都有維護一個理想節點清單 preference list 以維持資料品質，如果節點是透過 LB 獲得的 request 但卻不再目標 key 的理想清單上，他便不會成為 coordinator 而是把該 request 轉發。
- 一致性管理的部分也有類似 quorum 的機制，可以分別對讀取 R 和寫入 W 操作設定必須訪問幾個 peer node 確認一致性後才可執行。
  - 但 Dynamo 較為看中效能勝過一致性，因此通常會採用 R + W < N 的設定。
  - 當一個節點收到寫入操作，他會透過 vector clock 打上版號並廣播給該 key 的 preference list 上的所有節點，一旦有至少 W-1 個節點回報成功後該操作便可視為成功。
