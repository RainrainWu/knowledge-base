# The Google File System
> [原始論文](https://dl.acm.org/doi/pdf/10.1145/945445.945450?casa_token=AbimwJhe4_kAAAAA:141_axCki6JZqBR3hXW7kD2l7Gxm1PTxEL3yUDspYHcDvKaMXIlHI1CsAdBdGAQcBFMqmxQN2-Kz)
###### tags: `fault tolerance`, ` scalability`, `data storage`, `clustered storage`

# 摘要
- 本文內容主要是講解他們如何實現一個適合 Data-Intensive Applications 的可擴展大型分散式檔案系統。
  - 這裡的 Data-Intensive Applications 指的是軟體中資料在各個面向上的複雜度，畢竟任何儲存服務如 Database 其底層仍舊是檔案的組成，而隨著情境的變化和流量的提升，在數量級、讀寫頻率、與結構改變上都有了新的挑戰。
- 同時透過在大量的評價設備上運行服務來實現局部容錯性。
- 最初的動機是因為透過觀察到目前軟體服務的工作量的性質，和先前做的假設明顯不同，因此需要重新檢視。

# 介紹
延續前一段對工作量與性質的重新檢視，這裡作者提出了幾個觀點：
  1. 任意的組件其實非常容易遇到失效問題，主要是因為整個系統是由大量平價的設備組成，同時又有非常多人在使用的關係，所以持續不斷的監控和局部的容錯性是不可或缺的。
  2. 許多檔案的體積相對於以往都大非常多，幾 GB 的檔案都很常見，和以往針對大量 KB 級別檔案的系統有所不同。
  3. 相對於直接覆蓋一個檔案，更多的情境是在現有檔案內容的尾部資加資料，而讀取時往往會是從頭到尾的線性讀取。
  4. 同時設計檔案系統的 interface 和應用程式的 API 可以帶來更高的靈活性，可以針對不同的需求部署不同設定的 File System。

# 設計概觀
## Introduction
這裡延續上一段的觀察內容提出更嚴謹的假設：
1. 軟體服務由大量的平價設備運作，持續監控、局部容錯和重啟都是必要的。
2. 系統中數 GB 的大型檔案並不少見，而且應該是最佳化的重點。
3. 讀取的工作主要分兩種
  - 一種是大規模的 streamming read，單次操作約是數百 KB 到數 MB 之間，且同個使用者的連續操作會集中在檔案的連續區域。
  - 一種是在檔案中的隨機位置讀取 KB 級別的資料，使用者端為了效能考量多半會將多次隨機讀取統整並排序後再批次操作。
4. 寫入的工作有不少的大型檔案 streamming write，每此操作資料的量級和讀取差不多，在寫入完成後資料就很少再被改動。也時候也會有針對當按哪隨機位置的寫入，但不是效能最佳化的重點。
5. 多個使用者或服務對同的檔案的併發寫入的情境還蠻頻繁的，而且可能會有使用者或服務也在同步讀取，因此需要高效率的妥善處理多個寫入來源的合併歸檔。
6. 相對於對單個讀取或寫入操作要快速回應，應該更重視背後大量資料的處理效率和正確性。

## Interface
新的 file system 提供的介面並沒有太大改變，仍是 create, delete, open, close 等等概念，但是新增了 snapshot 和 records append 兩種：
  - snapshot 可以以低成本的方式備份一個檔案或一整個資料夾路徑。
  - record append 可允許多個使用者或服務對同個檔案尾部併發寫入新資料，同時保障個別操作的原子性，這對於整合郭的來源的寫入操作至關重要。

## Architecture
- Google file system 的 cluster 由一個 master 和多個 chunkservers 組成，通常每個節點都是個執行在平價的 Linux Server 上的 user-level process。
  - 每個檔案都會被切分為固定大小的 chunk，每個 chunk 在被創造時會由 master 配給一個 globally unique 的 64-bit id。
  - 而為了可靠性考量，每個 chunk 會在多個 chunkservers 上存有備份版本，而 chunkservers 可以像對待 Linux file 一樣進行 read 和 write 操作
  - master 節點主要負責管理整個叢集的 metadata，包含命名空間、存取權限、目前每個 chunk 的位置和 migrate 流程等等，同時透過 heartbeat 機制收集每個 chunkserver 的狀態。
  - 而使用者端主要就是透過使用實作了 Google file system 的 interface 來對目標檔案讀取與寫入的。
  - 而在使用者端與 chunkserver 身上都沒有 cache 機制，因為 streamming 操作的檔案往往非常大，cache 的效益並不高，反觀沒有 cache 後自然也就沒有了一致性的問題。
<img width="1472" alt="截圖 2021-05-21 下午6 57 25" src="https://user-images.githubusercontent.com/26277801/119127053-66428c80-ba66-11eb-8439-ba02a53f0b7f.png">

> [圖片來源為原始論文](https://dl.acm.org/doi/pdf/10.1145/945445.945450?casa_token=yOUt9xK1agUAAAAA:67h6kd4g-Uownn5b2Bbu2AS-SZSi5Cj1_jyXG-VQnfrtKEBChax1KOMF54c8UkzFu48b7E8F-Gei)

## Single Master
- 在一個叢集中永遠只會有一個 master 節點，這可以確保集中管理所有叢集內的資訊。
- 但就需要留意 master 節點中 metadata 讀取寫入是否成為效能瓶頸，因此讀取檔案的工作應該完全移交 chunkserver 執行。

## Chunk Size
- Google file system 的預設 chunk size 預設是 64 MB，比其他多數的 file system 都還要多。
- 由於採用 lazy space allocation 的關係，會先把寫入資料儲存在 RAM 中一直到其體積達到一定水準後，才配置一個物理上的儲存空間來寫入，這可以避免 internal fragmentation 的問題。
- 採用一個大的 chunk size 有個明顯的好處：
  1. 使用者須操作的檔案內容很大機率會在同一台 chunkserver 上，如此一來 master 有更高機率只要找尋單一個 chunk 所在的 chunkserver 即可。
  2. 從另方面來看使用者也有很高機率只需要對一台 chunkserver 發送請求即可，也可以維持一個長時間的連結不斷重複使用，因為他所需要操作的資料都在同一個 chunk 中。
  3. 更大的 chunk 體積意味著更少的 chunk 數量，master node 所需維護的 metadata 也隨之更少。
- 不過大的 chunk size 也有其問題：
  1. 當一個檔案所有的資料都在同個 chunk 時，若同意時間有很多個使用者要存取他，就會導致單一 chunk 的負載過高。但實際上他們的軟體服務更多時候是在存取多 chunk 的大型檔案。
  2. 之前第一次發生 chunk overload 的原因是因為針對 job queue 中的任務批次處理，然而其中就有大量其他服務啟動時針對同個檔案的操作，這問題最後透過提高數個熱門檔案的 replication factor 以及錯開相關服務的啟動時間來解決。
  3. 另個潛在的解決方案是讓使用者或其他服務彼此之間來相互詢問是否有剛剛存取到的 chunk，若有的話便可相互讀取分享。

## Metadata
- master node 所維護的 metadata 主要有三種
  1. 檔案和 chunk 的命名空間
  2. 檔案和其所在的 chunk 
  3. 每個 chunk 的備份副本所在的 chunkserver
- 前兩個檔案會透過 operation log 的機制來實現狀態正確性和可靠性，並可以備份到另一台遠端設備上。
- 對於備份所在位置並不會永久儲存，而是會在 master 啟動、新的 chunkserver 節點加入或是新的 chunk 創建時詢問各個 chunkserver 所持有的 chunk 來更新資訊。

### In-Memory Data Structures
- master node 的 metadata 主要是儲存在 RAM 內的，這允許他週期性的掃描所有資料也不會有太大負擔，同時也方便進行 garbage collection、metadata replication 等等。
- 然而採用這方法的缺點正是叢集大小和節點數量會受限於 master node 的 RAM 空間，這問題可以透過 prefix compression 有效緩解，就算要增加更多 RAM 的成本也非常低。

### Chunk Locations
- master node 並不會儲存每個 chunk 的備份副本位置，但是可以透過 heartbeat 機制和控制每個 chunk 新創建的流程來持續保存最新資訊。
- 作者提到最初他們也嘗試採用和前兩者一樣的 persistant storage 機制，但由於 chunkserver 發生改名、改設定檔的頻率不低，甚至發生單點錯誤導致下線或重啟，所以 polling 的方式有更高的可靠性。

### Operation Log
- 主要用途是留存對 metadata 更動的歷史紀錄，除了作為永久儲存的核心資料外，也是紀錄了併發操作先後順序的時間線。
- 具體實現類似於近代 RDBMS 的 write-ahead log，同樣有累極多筆更動或是間隔固定時間再一次寫入 Disk 來減緩 IO 壓力的優化方式。
- 同時也有 checkpoint 機制，在 log 體積超過一定限制後直接對所有歷史操作後的資料狀態進行儲存備份，並清空整理過的 log 以避免 log 無限制增長。
  - 雖說建立 checkpoint 等級的備份需要時間，但這並不會影響到服務繼續對 operation log 寫入的功能。

## Consistency Model
### Guarantees by GFS
- 檔案系統的命名空間操作（比如檔案改名、創建檔案）都必須是 atomic 的，也就是要嗎操作全部完成更新到新的命名空間，要嘛就全部回滾，這是透過 master node 中 namespace lock 來實現的。
- 每個檔案區域在受到改動後的狀態基本上可分為三個層級：
  1. defined: 資料仍維持的一致性且所有使用者能看到其所做的更動，對於沒有併發提交的更動會是這種狀況。
  2. consistent: 當同一時間有多個改動併發提交時，所有使用者仍然能看到所有改動後的最終狀態，但可能無法得知究竟是做了哪些更動。
  3. inconsistent: 如果一個更動遭遇錯誤的話就會是這狀態，不同使用者會看到不同的資料。
- Google file system 主要提供了 write 和 record append 兩種資料改動，前者是將資料直接覆蓋寫入在指定的檔案內容位置，後者是自動將新資料皆欲在檔案尾部寫下去。
- 對於同個 chunk 的多個備份副本，master node 會發布同樣先後順序得更動操作，並用版本號來檢驗 chunk 是否過期或沒更新到最新的更動內容，若發現毀損則會立即動用另個合法的副本來更新。

### Implications for Applications
- Google file system 透過上述對於操作的基本設計（比如 record append, checkpoint 等等）就已經能實現寬鬆的一致性。

# SYSTEM INTERACTIONS
- 這個系統中很大的一個關鍵就是要盡可能降低 master 的工作量負擔，這段會講一些 master node, chunkserver, client 在互動上的設計。

## Leases and Mutation Order
- mutation 是可以被 apply 到任一個 chunk 副本上的，對此使用到 lease 來確保 mutation 之間的順序一致。
  - master node 首先會選擇一個 chunk 眾多副本中的其中一個作為 primary，由他來提取對於該 chunk 的 mutation，而其他副本必須依循 primary 所編排 mutation 順序。
  - 而一旦 primary 成功的開始 apply mutation 後便可以透過 Heartbeat 機制持續的和 master node 要求延續授權。
  - 即使 master node 和當前 primary 失去聯繫，他也能立即選用另個副本做為 primary，之後的 mutation 順序便換成以新的 primary 為主。
  - 詳細流程圖可參考原始論文。
- 當一個 mutation 的體積過於龐大或是有橫跨到 chunk 的邊界的話，client 端會負責將其轉換為多個獨立步驟。
  - 但若是由很多個來源併發寫入的話，可能會導致彼此的多個步驟相互穿插，而形成上述 consistent 但並非 defined 的狀態。

## Data Flow
- 這部分的設計期望是盡可能善用各個節點的 network bandwidth，並避免壅塞和延遲問題。
  - 因此在傳輸上，資料是以線性的方式一個接個一個的更新到所有 chunk servers 上的，也久是說他的拓璞形狀是個鏈狀，而不是其他如 tree 或是 star 的結構。
  - 而為了迴避掉高延遲的傳輸路徑，每個 chunk server 在收到新資料時，都會選擇離自己最近且尚未收到新資料的節點進行傳送。其中對於距離遠近的判定是用 IP 來計算的。

## Atomic Record Appends
- 相對於直接寫入需要指定檔案中的 offset 位置，record append 只要指定檔案即可。
  - google file system 是採用 at-least-once 的交付機制，所以寫入至少會被執行一次。
- 然而這對於 client 來說就需要一些更複雜的實作：
  - 比如當這次 record append 會超出 chunk 大小時，chunk 會回覆給 client 要求在下個 chunk 重新操作，同時為了避免其端的操作發生，每次的操作大小應會被嚴格限制在四分之一 max chunk size 以下。
  - 若有任一個 chunk 副本的操作失敗，client 端都會在重新寫入一次，這導致可能有 chunk 因此持有重複的寫入資料。

## Snapshot
- 這個設計的目的是希望在盡可能不干擾當前的 mutation 執行的情況下建立出檔案或者個資料節路逕的備份副本。
  - 在執行一個 file 的 snapshot 前，master node 會撤回當前與該 file 相關的所有 chunk 中 primary 副本的 lease，這使得所有 client 端在找不到當前 primary 的情況下會主動聯繫 master node，這便讓 master node 有個機會來暫時阻擋更動並創建 snapshot。

# MASTER OPERATION
- master node 的主要工作包含 namespace 管理、chunk 生命週期管理、工作量負載平衡等等。

## Namespace Management and Locking
- master node 中的 metadata 和每個檔案所在的 full path 是一對一對應的，並且都帶有自己的 read-write lock。
- 每次要進行操作時，必須持有目標檔案路徑的 write lock 與其所有父路徑的 read lock，因為父路徑的 read lock 已經足以防止該路徑遭到刪除或重新命名。
- 其中 read lock 彼此之間不會 block，因此可以容許在同個路徑下的併發創建檔案。

## Replica Placement
- 將資料可靠性最佳化，並依照不同機器間的網路品質調整策略，比如同個機架上伺服器間的有線網路比跨機房的網路還要快，但同時也要顧慮整個機架毀損的問題。

## Creation, Re-replication, Rebalancing
- 再決定要把新的 chunk replica 放在何處時主要考慮三個因素
  - 是否有機器目前的硬碟空間用量過低，會傾向在硬碟用量較低的節點上創建
  - 每個伺服器當前的 chunk replica 創建數量，因為在創建的過程中會有大量的寫入因而壓縮運行服務可用的資源
  - ˋ否也能有物理上甚至地理位置上的分散性，比如不同機架與不同機房
- 一旦一個 chunk 的 replica 數量低於使用者設置的目標 master node 就會指揮選上的 chunk server 去複製，但多個複製機制之間也會有優先級，比如最近頻繁使用的活躍檔案就會比已經刪除的檔案更快被處理
- 同時 master node 也會控制每個 cluster 和 chunk server 同一時間執行的複製工作數量，以避免影響運作效能

## Garbage Collection
- Google file system 在垃圾回收上是採取惰性策略的，被刪除的檔案並不會被馬上回收空間，而是者有在特定的時刻才進行所有 file 和 chunk 的垃圾回收。

### Mechanism
- 當檔案被刪除時，他會被重新命名為一個隱藏的檔案名稱並帶有一個刪除的時間戳。
  - 當 master node 進行例行性的檔案掃描時，若隱藏名稱的檔案已經超過一定時間了他就會被回收掉（預設是三天）。
  - 在被刪除都可以透過重新命名為一個正常的檔案名稱來快速復原。
- 用於檢測 orphaned chunks 的機制也十分相似，透過 chunk server 和 master node 之間的 hearbeat 來比對資訊。
  - 當 chunk server 得知自身持有一個不在 master node metadata 中的 chunk 時，便可以自行刪除它。
 
