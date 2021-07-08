# Scheduling
- Go scheduler 是 go runtime 下的一個功能，會一併打包入執行檔中。

## GPM Model
- 整個 Go 語言的任務排程大致上可分為三個角色，分別是 Goroutine(G), Processor(P) 和 Machine(M)。
  - Goroutine
    - 任務排程的最小單位．類似於 application-level 的輕量級 thread，程式進入點的 main 函式就是個 goroutine。
    - 相對於 os thread 需要對 core context switch，goroutine 只需要在事先和 os thread 匹配好的 machine 進行 context switch 即可，成本相對低很多。
  - Processor
    - 調度資源執行 Goroutine 的處理器，負責包含上下文管理、context switch 等等事務，每個 processor 會匹配一個 machine 來實現 os 操作的能力。
  - Machine
    - 可以視為 os thread，在程式最初執行時就會建立並和 processor 匹配，並在後續長時間重複使用。
  - LRQ & GRQ
    - 每個 Processor 都會有一個 Local Run Queue (LRQ) 儲存自己接下來要執行的 Goroutine，而 Global Run Queue (GRQ) 則是用於儲存還沒被安排的 Goroutine

## Cooperative Scheduling
- Go scheduler 是屬於 cooperative 的性質，而不是 preemptive 的，因此不會有 goroutine 主動搶佔資源的情況。
- 但這也意味著必須妥善規劃 goroutine 執行的公平性，以避免極端情況下的 starving 等等問題發生，主要有四種時機讓 scheduling 重新調度資源。
  1. 用 keyword `go` 創建新的 goroutine 時。
  2. garbage collection
  3. system call (file io, netwark req, ...)
  4. synchronization & orchestration (mutex, channel operation, ...)

## Goroutine States
- 就像 os thread 一樣， goroutine 也會依照自身情況分為三個狀態，讓 scheduler 依據這些情況作出理想的調度。
  - waiting
    - goroutine 當前由於 system call、mutex lock 等等原因無法繼續執行被迫等待，這通常也是拖垮效能的主因。
  - runable
    - goroutine 已經滿足繼續執行的必要條件正在等待資源執行。
  - executing
    - goroutine 已獲得資源，正被 processor 透過匹配的 machine 執行指令中。

## System Call

### Asynchronous
- 像是 network call 之類的非同步 system call 在很多現代的 os 都可以做到非同步的機制，避免浪費時間等待不知道什麼時候會收到的 response。
- Go scheduler 也在 application 做了這方面的優化，發起 network call 的 goroutine 會被放到名為 net poller 的特殊 queue 上，直到收到 response 後才會被放回 LRQ 或 GRQ 中。

### Synchronous
- 有些特定的 system call 如 file io, atomic database transaction 等等會強制 block 住 os thread，此時 processor 會選擇讓出當前的 machine 讓 goroutine 完成，並另外請求一個新的 os thread 來匹配作為新的 machine，一旦 block 住的 goroutine 完成後便會放回 LRQ 中。

## Work Stealing
- 當一個 Processor 的 LRQ 已經空了的時候，他會嘗試著從其他地方取得多餘的工作，取用順序邏輯如下。
```
runtime.schedule() {
    // only 1/61 of the time, check the global runnable queue for a G.
    // if not found, check the local queue.
    // if not found,
    //     try to steal from other Ps.
    //     if not, check the global runnable queue.
    //     if not found, poll network.
}
```

## References
- [Scheduling In Go : Part II - Go Scheduler](https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part2.html)
