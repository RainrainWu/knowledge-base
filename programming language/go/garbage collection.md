# Garbage Collection
- GC 主要是管理儲存空間的重要任務，包含釋出不必要的空間以及保留仍在使用的部分。
- Go 目前的實作主要是透過 non-generational concurrent tri-color mark and sweep collector。

## Collector Behavior
- Go 的 garbage collector 在做 GC 時主要會有三個步驟。
  - Mark Setup - STW
    - 這步驟開始時會啟用寫入屏障 Write Barrier 阻止所有儲存空間的操作，所有的 goroutine 都必須停止以保持儲存空間的穩定性，這也就是所謂的 Stop The World (STW) 以及他所造成的後遺症。
    - 需要觀察適合的時機做 STW，多半是進行 function call 的時候，通常耗費 10 - 30 ms。
    - 但換句話說，如果是 data intensive 的計算比如大型 for loop 就可能導致等不到 STW 時機點的問題，這在 Go 1.14 版本後有新的 preemptive 機制來應對。
  - Marking - Concurrent
    - 這步驟本身也是 goroutines，並且會在開始時取用 25% 的 CPU 資源來執行，也就是說在 8 個 vCPU 的裝置上，會有 2 個 processor 被拿來做 marking 的工作。
    - 這些 goroutines 會併發的去 traverse 其他 goroutine stack 和 global heap 中正在使用的記憶體空間，並標記需要回收的資源。
    - 若次步驟因為 goroutine 過多或是寫入操作過於頻繁等等原因而執行過於緩慢，garbage collector 會放緩 allocation 的速度病使用更多的 application goroutine 來進行 mark assist 提升效率。
  - Mark Termination - STW
    - 在這個階段會把寫入屏障 Write Barrier 給關閉，並啟動各式各樣的清理工作，這段期間同樣是處于 STW 狀態的，平均耗費 60 - 90 ms。

## Configuration
- 針對 GC 的操作也有一些細微的設置可以調整，用來進一步的最佳化執行過程。
  - GC Percentage
    - 可以用來設置當 memory allocation 達到當前使用中空間的多少倍時，會觸發下一次的 GC。
    - 預設是 100，也就是多一倍。假設最近一次 GC 後仍有 200 MB 的 memory 正在使用中，那麼達到 400 ＭB 時就會觸發下一次 GC。

