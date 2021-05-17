# Cassandra
- 一個側重於大量寫入情境的效能，且有很強擴展性的分布式 NoSQL 資料庫，其在效能和可用性上有所突破的一大關鍵就是透過 timestamp 的先後順序邏輯擺脫了 lock 的機制。

## 資料儲存結構
- Cassandra 齊資料儲存結構類似於一個稀疏矩陣，仍然是建立於 table schema 之上的，但和其他使用 table schema 的 RDBMS 差別是他允許每筆紀錄只儲存其中幾個欄位的值。
  - 同時 Cassandra 在儲存上更近似於 row-based 的行為，因此在對單筆紀錄查詢時有較好的表現，但若要對大量紀錄的同個欄位做 aggregation 就會顯得苦手，更甚至在叢集化後資料分散在多個節點上會耗費更多資源。
  - [Row vs Column Oriented Databases](https://dataschool.com/data-modeling-101/row-vs-column-oriented-databases/)

### Partition
- 每筆資料都都應該帶有一個 partition key，cassandra 會依據這個欄位進行 hash 後決定他該放在 hash ring 哪個位置中。
  - 在叢集化後有多節點的情況下，會由各個節點分別負責一部份區段 hash ring 的儲存責任，若有做資料冗余的話可能會有兩個以上的節點儲存同一區段的資料，在應對單節點失效風險的情況下也因此有了同步上不一致性的問題。
  - 在 ver 1.2 之前是透過各個實體節點負責一段連續區段 hash ring 的資料儲存，同時依據 replecation factor 的參數決定區段重疊備份的範圍。
    <img width="1051" alt="截圖 2021-05-17 下午3 01 43" src="https://user-images.githubusercontent.com/26277801/118445274-cfea3000-b720-11eb-8bc7-281d76c54aef.png">
  - 在 ver 1.2 後因為要滿足切分為更多區段的需求，無限制的擴張實體節點成本相當高，於是讓單一實體可以儲存多個 virtual node，且其在 hash ring 上的位置是隨機的不需要呈連續性。
  - [Documentation Apache Cassandra](https://cassandra.apache.org/doc/latest/architecture/dynamo.html)

## Cassandra Query Language (CQL)
> 資料儲存上的結構限制也反映在語法的結構和條件上。
- 其中一個很關鍵的欄位是 partition key，因為對這欄位的 hash 結果會決定這筆資料被儲存在 hash ring 中的哪個 slot，並以此對應要去從集中的哪個節點的哪個里位置找資料。
  - 若不提供 partition key 則必須加上 `ALLOW FILTERING` 允許以其他欄位做篩選，相對而言就需要讀取更多資料損失些效能。
- 而同個 partition key 內的多筆資料是依靠著 clustering key 來進行有序排列的，因此若在 clustering key 上使用 `ORDER BY`, `RANGE SCAN` 操作時基本上可以省去排序的時間，反之其他欄位就需要計算資源去額外處理排序。

## 資料寫入
<img width="796" alt="截圖 2021-05-17 下午4 34 03" src="https://user-images.githubusercontent.com/26277801/118458834-b56a8380-b72d-11eb-87cc-97a94f452ecf.png">

1. 為了能實現遭遇意外下線後資料回復的能力，cassandra 也採用了如 RDBMS 中常見的 redo log 機制，只是它叫做 commit log，一樣是在任何其他操作前先寫入到 commit log。
2. 第二步是將資料戴上當時的 timestamp 直接寫入至 RAM 中維護的 Memtables，寫入期間不需要對欄位上鎖，因為只需要找 timestamp 最新的資料就可，也可以在短時間內保存新寫入資料在 RAM 中提供快速查詢，這是 cassandra 在大量寫入的情境中效能亮眼的一大主因。
  - 不過若使用到 `IF NOT EXISTS`, `WHERE {KEY} = {VALUE}` 之類的條件判斷 clause，則仍需要把先前的資料都搜集到才能判斷寫入操作，且往往需要從硬碟中讀取才能獲得最完整的資料，那了以上的優勢就不復存在了。
3. cassandra 同時也會在 RAM 中維護一個 Row Caches 專門提供高搜尋頻率的資料儲存的，對於相似於 row-based 的儲存結構，當一筆資料有個新的欄位被寫入時原本的 cache 就已經過期了，因此若檢查到正在寫入的資料存在 Row Caches 中就立刻讓他失效。
  - 若要追求極致效能的話，完成這步驟後就可以回覆 success 了，但因為此時尚未寫入硬碟，所以可能有些回覆 success 的資料無法做災難還原。
4. 資料會由 cassandra 內部機制批量寫回硬碟中，多半是 asynchronous 的以避免阻塞，同時若有需要從其他節點取的資料那自身就會成為 coordinator 的角色並可能需要做一些錯誤處理。
- [Cassandra 機制詳解](https://www.infoq.cn/article/j0mfq1cntskbk5rbdpvl)

### Compaction
- 當 Memtables 的體積過大，或是定時性的作業機制都會觸發寫入硬碟中 SSTable 的操作，這個行為和其他檔案的直接覆蓋有許多不同，在 cassandra 中被稱為 compaction。
- 由於 Memtable 中的資料是由外部直接寫入的，他可能是對於已經存在 SSTable 中的紀錄新增了一個之前沒有的欄位，那這時候就不能直接以新版覆蓋舊版，而是要採用 Merge 把兩個 tables 整理在一起，完成之後就會刪去舊的 tables。

## 資料查詢
1. 首先檢查專門存放高頻率搜尋數據的 Row Caches，若找到的話就可直接返回。
2. 其次則是查詢 Key Caches，這個部分是用來儲存 Memtables 和 SSTables 中每個 partition key 對應的位置的，將可以加快接下來的資料定位任務。
3. 接著是先查看 Memtables 裡對應的資料，因為這裡的資料都會比較新，也在 RAM 中查詢比較快，都沒有的話才會進入 Disk 中的 SSTables 查找。
4. 找到之後如果 Row Caches 允許存入這筆資料的話可順便存入，方便下次快速取用。

## Clustering
- 由於資料是基於 partition key 來分隔儲存的，搬動 partition 的物理位置或是複製其資料內容的成本很低，因此享有很高的擴展性。
- 在叢集中多個 nodes 或 virtual nodes 之間並沒有 master-slave 的位階關係，每個節點都是平等的，可以接收寫入操作也可回覆自身持有的資料。
- 雖說 cassandra 在分布式系統的 CAP 悖論中直觀上選擇了 AP edge，意味著在一致性上需有所退讓，但他仍有提供 tunable consistency 讓使用者視情況調整對一致性的要求。
  - 其一致性的等級是依照每次操作所需訪問的節點數量來決定的，比如 `ANY`, `ONE`, `TWO`, `QUORUM`, `ALL` 等等，比如選用 `TWO` 的話就表示這次操作需要訪問兩個節點確認有相同狀態後才可執行。
  - 而讀取的操作和寫入的操作也是可以分開設定的，比如對於寫入遠大於讀取且資料並沒有嚴謹的的正確性要求，但希望讀取時總能讀到最新資料時，可以對寫入僅設定 `ONE` 但讀取設定 `ALL` 以獲得更理想的性能。
