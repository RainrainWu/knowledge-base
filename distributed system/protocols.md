# Protocols
- 雖說分散式系統享有了 high availability, high throughput 的優勢，但在面對需要 consistency 的情況時 stale data, split-brain 等等的問題。
- 一個妥善設計的 consensus protocol 有助於分散式系統那個節點的溝通與資訊同步。

## Raft
- 屬於 leader-based protocol 的一種，在許多近代分散式系統廣為採用，比如 etcd cluster。
- 其同步原理類似於 event sourcing，會把每個節點視為狀態機，而其傳遞與儲存的 operation log 則是對其操作的指令，只要將所有 operation logs 依序執行完就會獲得相同的最新狀態。
- 所有寫入請求都是由 leader node 接收後再擴散至其他 follower node。
- 為了維持共識狀態和 leader-follower 架構運作，Raft 有兩個關鍵的 RPC 操作，分別是 AppendEntries RPC 和 RequestVote RPC。
  - AppendEntries RPC
    - 主要用於擴散新的 operation log，對於一個 follower 一次只會同步一筆 operation log，原子性的資料同步效率雖然比較低，但也因此不用顧慮部分成功部分失敗的情況，總之出問題就是在傳一次 overwrite 就好。
    - 基本上一個寫入行為要成功收到半數以上的 follower nodes 回報成功後才能被視為成功寫入，但必要的話也可以調整這個參數。
    - 就算沒有 operation log 要更新也會作為 heartbeat 發送，讓 follower 知道 leader 還活著。
  - RequestVote RPC
    - 一旦 follower node 超過一定期限沒有收到 leader node 的 heartbeat，他就會當作 leader 已經死了並通知其他 follower 發起投票。
    - 投票行為是帶有單調遞增的版本號的，因此就算原本的 leader 因網路隔離問題一陣子後又加回來，也會因為版本較舊被視為過期而降級為 follower。
    - 假設當前版本 t 的 leader 掛了，第一個觸發 heartbeat 超時機制的 follower 就會向其他 followers 發出 t + 1 版本的 RequestVote RPC，若半數以上都同意他便會他便會成為新的 leader node 並開始擴散 operation log。
    - 盡可能將各個 follower node 的 heartbeat 超時時間設置的不一樣，避免一瞬間有大量的 RequestVote RPC、跳了好多個版本號的 leader。
