# Database Table Schema
- Schema 規範了資料在資料庫中儲存的結構，對商業邏輯操作複雜度和效能都有很大影響，因此在 System Design 必須重視 Data Model 的階段並思考到 Schema 如何設計的問題。
- 除非有硬性限制，否則盡可能避免直接讓 Table Schema 相互繼承導致 parent-child 的樹狀結構，建議以耦合較低的 Foreign Key 實作。
- 雖說對於 Schema 往往希望能做到 3NF 以上的正規化，但有些特例情況也是可以保留些彈性來換取顯著的效能提升，比如
  - 保存篇貼文按讚人數的 Redundancy 欄位，省去每次都要用 COUNT 去計算的時間
