# PostgreSQL 中常用的 Index
- PostgreSQL 除了預設使用 Primary Key 來建構 Heap Table 以外，也能讓使用者是情況添加 Secondary Index。
- 內建有提供幾種實用的 Index 可供選擇，依據使用的情境選擇適合的 Index 對效能改善是非常重要的。
- 寫筆記時的最新版本是 PostgreSQL 13，就以這版本來紀錄。

## B-tree
- 原理近似於平衡搜尋樹，可以用於排序，也支援大於、小於、等於的比較操作，但僅限於內建的基本資料型別。
- 結構上需要一個儲存 MetaData 的 meta page，以及作為搜尋樹根節點的 root page。
- 隨著資料量增加一個 root page 可能不夠儲存，因而會向下分支出若干層 branch page 與葉節點的 leaf page。

### 參考資料
- [PostgreSQL 官方文件](https://www.postgresql.org/docs/current/btree-intro.html)
- [深入浅出PostgreSQL B-Tree索引结构](https://github.com/digoal/blog/blob/master/201605/20160528_01.md?spm=a2c6h.12873639.0.0.45131bffpkfqz3&file=20160528_01.md)

## GiST
- 全名是 Generalized Search Tree，概念上也是平衡搜尋樹的一種，但在可接受的資料型別以及比較方法上有更多彈性，比如：
  - 幾何圖形是否相交
  - 經緯度座標上到市中心的距離由近至遠排序

### 參考資料
- [PostgreSQL 官方文件](https://www.postgresql.org/docs/current/gist-intro.html)
- [PostgreSQL 百亿地理位置数据 近邻查询性能](https://github.com/digoal/blog/blob/master/201601/20160119_01.md?spm=a2c6h.12873639.0.0.45131bffpkfqz3&file=20160119_01.md)

## SP-GiST
- 全名是 Space-partitioned Generalized Search Tree，相對於 GiST 更進一步的聚焦在抽象空間切分方面的搜索功能，比如：
  - 地理空間中可能以垂直水平切分成四個區域，而形成 Quad Tree
  - 電話號碼和 IP 中可透過帶有區域性質的數字來做分群，形成 Trie
  - 而這些資料結構往往都是 Unbalance Tree，GiST 原有的設計並不擅長處理

### 參考資料
- [PostgreSQL 官方文件](https://www.postgresql.org/docs/current/spgist-intro.html)
- [SP-Gist -- A new indexing framework for PostgreSQL](https://github.com/digoal/blog/blob/master/201706/20170627_01_pdf_001.pdf?spm=a2c6h.12873639.0.0.45131bffpkfqz3&file=20170627_01_pdf_001.pdf)

## GIN
- 全名稱是 Generalized Inverted Index，原理就是倒排索引，專門處理一些在 composite value 中搜尋 key item 的情況，比如：
  - 搜尋有特定關鍵字的文本
  - 搜尋有特定數值的數組

### 參考資料
- [PostgreSQL 官方文件](https://www.postgresql.org/docs/current/gin-builtin-opclasses.html)
- [PostgreSQL GIN索引实现原理](https://github.com/digoal/blog/blob/master/201702/20170204_01.md?spm=a2c6h.12873639.0.0.45131bffpkfqz3&file=20170204_01.md)

## BRIN
- 全名稱是 Block Range Index，是依據每個資料儲存區塊的統計資訊，比如：
  - 比如時間戳的上下界
  - 地理位置的 Bounding Box
- 實際使用上適用於資料內容和物理儲存位置高度相關的情況，可以快速聚焦較小範圍的資料來搜尋，而不用花時間掃描其他幾乎不相關的資料，比如
  - 按照時間先後一筆一筆寫入的商品銷售紀錄
  - 按照郵遞區號分群儲存的房租資料
- BRIN Index 的精準度可以透過指定每一個 range 可包含的紀錄數量來決定，range 越小統計資料就越精準，但也需要更多的空間來儲存更多的 Index 統計資料。

### 參考資料
- [PostgreSQL](https://www.postgresql.org/docs/current/brin-intro.html)
- [PostgreSQL 9.5 new feature - BRIN (block range index) index](https://github.com/digoal/blog/blob/master/201504/20150419_01.md?spm=a2c6h.12873639.0.0.45131bffpkfqz3&file=20150419_01.md)

## hash
- 透過 hash 後的數值來定位正確的紀錄，適合處理 Value 非常長的欄位，但也只能支援完全相等的查詢，沒辦法用於大於小於的比較。

