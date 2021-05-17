# Cassandra

## 資料儲存結構
- Cassandra 齊資料儲存結構類似於一個稀疏矩陣，仍然是建立於 table schema 之上的，但和其他使用 table schema 的 RDBMS 差別是他允許每筆紀錄只儲存其中幾個欄位的值。
  - 同時 Cassandra 在儲存上更近似於 row-based 的行為，因此在對單筆紀錄查詢時有較好的表現，但若要對大量紀錄的同個欄位做 aggregation 就會顯得苦手，更甚至在叢集化後資料分散在多個節點上會耗費更多資源。
  - [Row vs Column Oriented Databases](https://dataschool.com/data-modeling-101/row-vs-column-oriented-databases/)

## Cassandra Query Language (CQL)
> 資料儲存上的結構限制也反映在操作語法的結構和條件上。
- 其中一個很關鍵的欄位是 Partition key，因為對這欄位的 hash 結果會決定這筆資料被儲存在 hash ring 中的哪個 slot，並以此對應要去從集中的哪個節點的哪個里位置找資料。
  - 若不提供 Partition key 的話則必須加上 `ALLOW FILTERING` 允許以其他欄位數值做篩選，但相對而言就需要讀取更多資料而損失些效能。
