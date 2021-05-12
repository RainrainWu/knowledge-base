# RDBMS 對 ACID 的要求
- 在對資料庫執行 transaction 操作時，為了確保資料在執行出錯、網路問題或斷電等等意外情境下的合法性，而在幾個特性和行為上有所要求。
- 但每個資料庫對 ACID 的定義不盡相同，就算官方文件宣稱自己滿足 ACID 原則也還是要看一下具體的行為描述。

## Atomic
- 一筆 transaction 中可包含多個 SQL command，但同個 transaction 中的 SQL command 只能全部都依序執行完畢，或者全部都不執行。
- 遭遇斷電之類的意外時，未被 commit 的 transaction 全數 rollback，需修復資料時資料庫必須以 transaction 為基本單位逐一修復。
- 在幾乎絕大多數的商業系統中，多個欄位的 atomic 性質都是必須的，比如轉帳時會牽扯到兩個帳戶的金額。

## Consistency
- 資料庫必須從一個正確的狀態，轉移到下一個正確的狀態，比如：
  - 使用者付錢買票，即使執行出錯也必須停留在「付錢且買到票」或是「沒付錢也沒買到票」的情況，不能是「付錢卻沒有票」或是「沒付錢卻拿到票」的情況。

## Isolation
- 不同 transaction 執行時不應該相互影響，也只能存取發起 transaction 時能夠存取的資料。
- Isolation 有多個層級可調整，理論上越嚴謹的隔離會犧牲越多效能，依據情況選擇合適的層級。
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
- 當多筆 transaction 併發執行時，可能會因為其執行完成的先後順序而有不一致的情況

### Isolation Level
- 依據是否能完全避免上述互相干擾的狀況，資料庫的隔離層級可分為四個層級。
- PostgreSQL 預設的隔離層級是 Read committed。

|Isolation Level|Dirty Read|Nonrepeatable Read|Phantom Read|Serialization Anomaly|
|-|-|-|-|-|
|Read uncommitted|Allowed, but not in PG|Possible|Possible|Possible|
|Read committed|Not possible|Possible|Possible|Possible|
|Repeatable read|Not possible|Not possible|Allowed, but not in PG|Possible|
|Serializable|Not possible|Not possible|Not possible|Not possible|

## Durability
- 一筆 transaction 一旦被 commit 後，除非儲存空間受損否則將永遠不會消失。
