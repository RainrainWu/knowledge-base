# Data Consistency

## Conflict-free Replicated Data Type (CRDT)
- 在分散式的系統中，若要能容忍部分節點失效同時對於每次存取又要盡可能正常響應的情況下，各個節點勢必得要有一定程度的獨立運作能力。
- 但是在這各自為政的情況下，每次對於資料的更新可能沒辦法即時同步到叢集中的每個節點，進而產生 consistency 的問題。
- CRDT 正式為此而被設計出來的一種資料 converge 機制，以 strong eventually consistency 為目標，在多個節點各自有資料更動時，最終仍然能夠和併入一個單一的結果。

### Commutative
- 如果對於資料的更動能滿足 commutative 的特性的話，CRDT 是一個很理想的解決方案，因為他從根本上解決了 race condition 的問題。
- 以四則運算來說：
  - 加法是 commutative 的：`a + b = b + a`
  - 減法是 commutative 的：`a + -b = -b + a`
  - 乘法是 commutative 的：`a * b = b * a`
  - 但除法就不是 commutative 的：`a / b != b / a`
- 但若以 NoSQL 常用儲存格式 JSON 的操作來說就複雜得多：
  - integer counter 的 increment, array 的 insert, object 的 delete 都是 commutative 的。
  - 但 object 的 set 就不是了，通常是最後一個 set 操作決定了最終 value，也就是 last-writer-wins (LWW) 的機制。
  - 因此 set 的操作盡可能僅限於不會有 concurrency 也不會有 conflict 的情況，比如設定常數參數。

### Usage
- CDN-edge data replication
- multi-master cluster replication

### Examples
- 即使每個節點收到不同的 concurrent 操作，仍然能透過 broadcast 來達到 strong eventually consistency。
  <img width="1274" alt="截圖 2021-07-17 下午11 25 54" src="https://user-images.githubusercontent.com/26277801/126041776-c528572b-732d-4c51-8692-722d35ff5ca8.png">

### References
- [CRDTs for Non Academics](https://www.youtube.com/watch?v=vBU70EjwGfw)
  - 影片中有對於 increment/decrement counter, version UUID 等等核心概念比較好的解釋。
  - 個別節點能夠獨立解決 conflict 問題的一大關鍵就是充足的 metadata，比如 version UUID, timestamp, GC version, dependency vector clock 等等。
