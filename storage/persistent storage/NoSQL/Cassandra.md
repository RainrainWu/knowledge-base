# Cassandra 

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

## 資料搜尋
> 資料儲存上的結構限制也反映在語法 Cassandra Query Language (CQL) 的結構和條件上。
- 其中一個很關鍵的欄位是 partition key，因為對這欄位的 hash 結果會決定這筆資料被儲存在 hash ring 中的哪個 slot，並以此對應要去從集中的哪個節點的哪個里位置找資料。
  - 若不提供 partition key 則必須加上 `ALLOW FILTERING` 允許以其他欄位數值做篩選，但相對而言就需要讀取更多資料而損失些效能。
- 而同個 partition key 內的多筆資料是依靠著 clustering key 來進行有序排列的，因此若在 clustering key 上使用 `ORDER BY`, `RANGE SCAN` 操作時基本上可以省去排序的時間，反之其他欄位就需要計算資源去額外處理排序。

## 資料寫入
