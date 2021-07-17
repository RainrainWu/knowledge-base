# Data Consistency

## Conflict-free Replicated Data Type (CRDT)
- 在分散式的系統中，若要能容忍部分節點失效同時對於每次存取又要盡可能正常響應的情況下，各個節點勢必得要有一定程度的獨立運作能力。
- 但是在這各自為政的情況下，每次對於資料的更新可能沒辦法即時同步到叢集中的每個節點，進而產生 consistency 的問題。
- CRDT 正式為此而被設計出來的一種資料 converge 機制，以 strong eventually consistency 為目標，在多個節點各自有資料更動時，最終仍然能夠和併入一個單一的結果。
- 如果對於資料的更動順序並非至關重要的話，CRDT 是一個很理想的解決方案。
