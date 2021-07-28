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
- Dynamo 首要目標就是 always writeable。
