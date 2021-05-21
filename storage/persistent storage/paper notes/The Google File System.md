# The Google File System
> [原始論文](https://dl.acm.org/doi/pdf/10.1145/945445.945450?casa_token=yOUt9xK1agUAAAAA:67h6kd4g-Uownn5b2Bbu2AS-SZSi5Cj1_jyXG-VQnfrtKEBChax1KOMF54c8UkzFu48b7E8F-Gei)
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
