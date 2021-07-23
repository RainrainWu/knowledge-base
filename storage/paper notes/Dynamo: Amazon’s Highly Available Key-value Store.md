# Dynamo: Amazon’s Highly Available Key-value Store
> [Origin Paper Link](https://dl.acm.org/doi/abs/10.1145/1323293.1294281?casa_token=Vhd9CIMdUUEAAAAA%3Aiz1gZeBIcCLgkzCxdoWZC4G2VWkMQdx62srpHJo18ZsT2o6RqBy-6MGsUEi8XGZ2LhqdRWGRBnS7)

## Context
- 在上萬節點規模的叢集上追求可靠性是非常有挑戰性的，因為節點失效和網路問題會持續的發生，並影響使用者的體驗。
  - 最理想的解決辦法不是想辦法避免他，而是直面著個問題並設法與他共存。

## Introduction
- 其中一個學到的領悟是一個系統的 reliability 和 scalibility 很大程度的依賴於如何儲存與管理 state 相關的資料。
  - 比如遇到網路波動、硬碟壞軌、或是洪水沖毀某個 data center 時都仍要讓使用者能放商品到購物車中。
- Amazon 的許多服務只需要透過 primary key 來獲取資料，比如商品訊息、賣家資訊等等。
  - 不過使用傳統的 RDB 會在 performance 和 scaling 上造成許多局限，因此 Dynamo 期望能提供一個同樣是基於 primarykey 來取的資料的全新解決方案。
- Dynamo 應用了許多常見提升 availibity 的方法，比如透過 consistent hashing 實現 partition 和 replication。
- 而 consistency 的需求則有使用到了 versioning 和 quorum-like 的多數共識門檻，以及 gossip based 的節點失效偵測機制。
- 簡單而言，Dynamo 實現了一個幾乎不需要手動維護，加增或刪除節點時也不須重新部署的 HA KV store。
