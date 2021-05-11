# 原理
- 資料庫在執行 CRUD 等等常務性操作時，並不會考量日後各種資料被查詢的情境，因此每一筆記錄多半是依照統一的規則儲存的。
- 而有時候特定的資料排列方式是能非常顯著的改善查找紀錄的效率的，因此特意使用額外空間維護其結構便是一種可行的方法。
- PostgreSQL 會自動以 Primary Key 來建立 B+Tree 實現 Heap Table，而使用者能自行決定額外加上的都是 Secondary Index。
- 但也因此使用 Index 時多半會有額外的物理儲存空間佔用，也因為每次更新資料也須同時更新 Index 所以會犧牲一些效率。
- 儘管在時間複雜度上有所改善，建立 Index 與更新 Index 的時間成本仍是與資料量成正比的，需先妥善估計並規劃時程。

# B+Tree
- 為了快速定位到目標資料，近代主流的 RDBMS 都會使用 Primary Key 建立 B+Tree 來提升搜索速度。
- B+Tree 的 Non-Leaf Page 主要是紀錄搜尋時的分支判斷，因為會被頻繁使用所以多半會一直待在 Cache 中。
- 而目前主流的兩種實現方式 Index-Organized Table 和 Heap Table 和 的差異主要是在 Leaf Page 所儲存的東西。

## Index-Organized Table
- 這個實現方式會直接把資料給存在 Leaf Page 中。
<img width="868" alt="截圖 2021-05-11 下午3 39 46" src="https://user-images.githubusercontent.com/26277801/117777384-2104aa80-b26f-11eb-805e-fde596ddd648.png">

### 優點
- 所有的紀錄已經依照 Primary Key 的順序排好，省去 Sort by Primary Key 的時間。
- 由於連續性的資料會連續的存放在一起，所以很適合做 Range Scan 相關的操作，比如一段時間內的營業分析報表。

### 缺點
- 所有 Create 新資料的操作都會集中在最右側的 Leaf Page 上，若是頻繁 Create 資料的情境容易頻繁被 Lock 拖慢效能。
- 由於每個 Leaf Node 儲存空間有限，若是頻繁 Insert/Delete 的話可能需要不斷觸發 Split Node 和 Merge Node 來調整。
- 為了維持資料基於 Primary Key 的順序，Split/Merge Node 會對資料移動其物理上的儲存位置，因此 Secondary Index 也只能儲存資料的 Primary Key 而不是物理位置。

## Heap Table
- 這個實現中 Leaf Page 會儲存資料的對應位置，不會直接儲存資料。
<img width="925" alt="截圖 2021-05-11 下午4 13 34" src="https://user-images.githubusercontent.com/26277801/117782057-db96ac00-b273-11eb-87d2-9f30c0af2a8a.png">

### 優點
- Create 新資料時可以選擇相對空閒的 Leaf Page 和 Data Page 寫入，避免過度集中一個 Page 導致 Blocking。
- Leaf Page 由於只存放資料的對應位置空間相對較小，觸發 Split/Merge Node 的需求也相對更低。
- Split/Merge Node 時不會改動到資料儲存的物理位置，因此 Secondary Index 可以考慮直接儲存資料的物理位置。

### 缺點
- 由於資料之間是無序排列的，所以 Range Scan 時需要存取幾乎所有的 Data Page 很吃重 Disk IO。
- 一般情境下處理商業邏輯為主的 OLTP 資料庫使用 Range Scan on Primary Key 的情境不多，但對於分析用或產報表的 OLAP 系統來說十分致命。

# Primary Key 選擇
- Primary Key 是預設被用於建立 B+Tree 的 Index，而如何選擇 Primary Key 主流有兩種方法。

## Natual Key
- 直接使用自然的 Unique Key 做 Primary Key，比如帳號、產品序號、電子郵件

### 優點
- 可以省掉建立 Secondary Index 的成本，比如即使不用產品序號做 Primary Key，多半也要對產品序號建立 Index。
- Natual Key 本身也有足夠的隨機性避免大量的 Create 集中在同個 Leaf Page 上而被 Block。

### 缺點
- 有些 Natual Key 可能長短不一，比如電子郵件有些就特別長，為了保險起見可能會預留較大空間而造成浪費。
- Natual Key 走可能失效，比如可以讓使用者透過帳號以外的方式登入時，帳號作為 Primary Key 就無法做出調整。

## Surrogate Key
- Surrogate Key 並沒有實際商業上的意義，只是人工設計出來專門用於邏輯判斷的。

### 優點
- 和商業邏輯完全解耦，能應對更多意料之外的情況。
- Primary Key 長度更加可控制，配置適當的空間避免浪費。

### 缺點
- 採用 Auto Increment 的 Primary Key 時，為了避免重複本身也是自帶一個 Lock。
- Auto Increment 的 Primary Key 理論上沒有隨機性，大量 Create 時會導致操作集中在最右邊的 Leaf Page 而頻繁被 Block。
- UUID 也是有可能 Conflict，只是機率很低而已。

# PostgreSQL 中常用的 Index
- Postgres 有提供幾種實用的 Index 可供選擇，依據使用的情境選擇適合的 Index 對效能改善是非常重要的。
- 寫筆記時的最新版本是 PostgreSQL 13，就以這版本來紀錄。

## B-tree
- 原理近似於平衡搜尋樹，可以用於排序，也支援大於、小於、等於的比較操作。
- 結構上需要一個儲存 MetaData 的 meta page，以及作為搜尋樹根節點的 root page。
- 隨著資料量增加一個 root page 可能不夠儲存，因而會向下分支出若干層 branch page 與葉節點的 leaf page。

### 參考資料
- [PostgreSQL 官方文件](https://www.postgresql.org/docs/current/btree-intro.html)
- [深入浅出PostgreSQL B-Tree索引结构](https://github.com/digoal/blog/blob/master/201605/20160528_01.md?spm=a2c6h.12873639.0.0.45131bffpkfqz3&file=20160528_01.md)

## GiST
- 全名是 Generalized Search Tree，概念上也是平衡搜尋樹的一種，但在可接受的資料型別以及比較方法上有更多彈性，比如：
  - 幾何圖形是否相交
  - 經緯度座標上到市中心的距離由近至遠排序
  - IP 位址是否在同個區段

### 參考資料
- [PostgreSQL 官方文件](https://www.postgresql.org/docs/current/gist-intro.html)
- [PostgreSQL 百亿地理位置数据 近邻查询性能](https://github.com/digoal/blog/blob/master/201601/20160119_01.md?spm=a2c6h.12873639.0.0.45131bffpkfqz3&file=20160119_01.md)

## hash
## GIN
## SP-GiST
## BRIN
