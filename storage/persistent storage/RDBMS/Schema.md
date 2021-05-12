# Database Table Schema
- Schema 規範了資料在資料庫中儲存的結構，對商業邏輯操作複雜度和效能都有很大影響，因此在 System Design 必須重視 Data Model 的階段並思考到 Schema 如何設計的問題。
- 除非有硬性限制，否則盡可能避免直接讓 Table Schema 相互繼承導致 parent-child 的樹狀結構，建議以耦合較低的 Foreign Key 實作。
- 雖說對於 Schema 往往希望能做到 3NF 以上的正規化，但有些特例情況也是可以保留些彈性來換取顯著的效能提升，比如
  - 保存篇貼文按讚人數的 Redundancy 欄位，省去每次都要用 COUNT 去計算的時間

## Anti-pattern
- 除了一些明顯違反正規化的 Schema 設計外，也有些常見的 Schema 設計問題是可以避免的。

### Smart Column
- 如 PostgreSQL 中儲存資料型別的一大亮點正是 JSON, XML, ARRAY 等等功能更有彈性的 Flexible Type。
- 但這些型別其實都是違反正規化中 1NF 的 atomic value 限制的，除了無法在讀取前明確得知儲存內容結構外，也可能會有查詢比較上的性能消耗，比如：
  - 想透過一個 JSON 的 key-value 來排序，但無法在讀取前知道是否所有 records 的欄位都有那個 Key。
  - 想透過一個 JSON 的 key-value 來找特定 row，便無法直接用 `=` 邏輯運算符來判斷，而要使用 `like` 之類關鍵字來做 Flexible Type 的判斷。

### Multi-Functional Table
- 有些 Table 在設計時做了過度的抽象化，想把 Data Model 結構相近但用途根本不同的紀錄都存在同一張 Table 中，比如貨物進出口紀錄、商品付款及退款紀錄。
- 一個明顯的判斷是 Table 中有多個欄位是空值，比如貨物進口紀錄的出口目的地欄位就都會是 NULL，商品退款紀錄的付款時間也都會是 NULL。
- 另個明顯的判斷是有個 Column 名稱和 `Type`, `Kind` 相關，需要先確認此欄位才能知道該筆 record 正確的使用方式。
- 這種 Schema 除了留有大量空直佔用儲存空間外，也容易因為同個 Table 存在多種使用邏輯讓 query planner 難以釐清效能最佳的 query plan。
