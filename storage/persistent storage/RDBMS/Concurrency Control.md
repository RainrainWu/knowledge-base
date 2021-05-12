## Concurrency Transactions
- 為了能有更高效率，資料庫會傾向同時間執行盡可能多的 Transactions，並輔以適當的隔離層級來確保資料安全性。
- 資料庫的隔離層級中最嚴格的就是 Serialize ，完全序列化確保每一筆 Transaction 逐一執行，而其他三個層級則是依據是否能妥善防止 transaction 間可能引發問題的相互干擾行為來劃分的。

### Read Phenomena
- 當資料庫允許多個 transaction 併發執行時，可能引發的互相干擾狀況，隨著隔離層級要求由低至高排列：

#### Dirty Read
- 一筆 transaction 執行時可以讀取到另一筆 transaction 尚未 commit 的資料。

#### Nonrepeatable Read
- 同一筆 transaction 中執行相同的 SQL command 多次讀取資料時，由於該欄位中途有被其他 transaction 提交改動，因此相同的 SQL command 讀取到的資料不相同。

#### Phantom Read
- 同一筆 transaction 中執行相同的 SQL command 多次讀取**資料集合**時，由於相關 table 中途有被其他 transaction 提交改動，因此相同的 SQL command 讀取到的**資料集合**即使同樣符合條件其內容也可能不同。
- 常見於一些 aggregation 操作，比如 SUM, AVERAGE, COUNT 等等。

#### Serialization Anomaly
- 當多筆 transaction 併發執行時，可能會因為其執行完成的先後順序而有不一致的情況。

### Isolation Level
- 依據是否能完全避免上述互相干擾的狀況，資料庫的隔離層級可分為四個層級。
- PostgreSQL 預設的隔離層級是 Read committed。

|Isolation Level|Dirty Read|Nonrepeatable Read|Phantom Read|Serialization Anomaly|
|-|-|-|-|-|
|Read uncommitted|Allowed, but not in PG|Possible|Possible|Possible|
|Read committed|Not possible|Possible|Possible|Possible|
|Repeatable read|Not possible|Not possible|Allowed, but not in PG|Possible|
|Serializable|Not possible|Not possible|Not possible|Not possible|

### Lock
- 要能控制資料能否被讀取的權限以實現 transaction 隔離，近代的 RDBMS 主要有 shared-exclusive lock 和 MVCC 兩種實現。
- PostgreSQL 是採用 MVCC。

#### Shared-exclusive Lock
- 資料庫中的每一筆紀錄都有自己的 SX Lock，S Lock 對應的是讀取權限可以發出很多把，X Lock 對應的是寫入權限只能發出一把。
- 每個 transaction 要存取對應紀錄時，都必須獲得該筆紀錄對應權限的 Lock 才准許操作，並且在 transaction 結束時歸還所有 Lock。
- 發放 X Lock 時該筆紀錄必須沒有其他發放出去的 S Lock 和 X Lock，而在一個 X Lock 尚未收回前也不能發放新的 S Lock 和 X Lock。

#### MVCC
- 從一筆紀錄被創建開始，所有的寫入改動都會是為其增加新的版本而不是直接修改資料，因此會有一個類似版本號的遞增數值來表示，而過舊的版本會被清理掉。
- 執行 transaction 時會依據其執行的時間點控制對應紀錄的可見性，也就是說該次 transaction 只能看見執行時間點最新版本的紀錄，其他更舊貨更新的版本都是不可見的。
- 也因為採用多版本控制的關係，所以並不會有 Read Lock，永遠能讀取到對應的版本。

#### 優缺點
- MVCC 由於要判斷哪個才是最新版本、相對資料的可見性與清除過舊的版本，因此會有更高的 CPU 和 Disk IO 開銷。
- MVCC 也得益於多版本不會導致讀取被 block 的優勢，在高流量情境下會有更好的效能表現，特別是 Read 操作更多的時候。
- Shared-exclusive Lock 由於同一筆資料只儲存一個實體，相對於儲存多個版本的 MVCC 更省空間。
- Shared-exclusive Lock 會有 read lock 需要管理，deadlock 在偵測上需要更加複雜的檢查且真的造成 deadlock 的機率會更高些。
