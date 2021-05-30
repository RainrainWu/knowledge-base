## Scheduling

### Global Interpreter Lock
- Python 有一把 GIL 用來確保同意時間只有一條 thread 在執行 python bytecode 來存取底層的 object，其背後的原因主要是因為 Python 的 memory management 機制並不是 thread-safe 的，需要避免 race condition。
  - 比如原本有個變數 a = 2，目前有兩個 thread 同時執行：
    - a += 4
    - a *= 2
  - 若兩個 thread 同時存取到 a = 2 的值，那麼最終 a 會是 6 還是 4 就會取決於 thread 執行的順序，這種隨機性發生的錯誤會有很嚴重的邏輯問題。
- GIL 會很大程度的限縮 Python 在多 CPU 的機器上運行的好處，不過 network I/O, 圖像處理等等並不需要取得 GIL 因此並不是每個 thread 都有阻塞的問題。
- 若要擺脫 GIL 的話可以使用 `multiprocessing` 套件，由於 process 之間的記憶體資源不會共享，也久不必要顧慮 thread-safe 的問題。

#### Reference
- [GlobalInterpreterLock](https://wiki.python.org/moin/GlobalInterpreterLock)
- [The Python GIL](https://python.land/python-concurrency/the-python-gil)

### Scheduling with GIL
- Python 3.2 後採用了新的 GIL 並沿用其主要機制至今，以下將會以新的 GIL 為主要討論對象。
  - 每個 thread 都有一個 `gil_drop_request` 的變數，當這個變數被設置為 1 時，執行中的 thread 就必須釋出當前持有的 GIL，並等待獲取到 GIL 的 thread 回應成功時自身在會進入待命週期。
  - 其餘待命中的 thread 都有各自的等待時間週期，一旦 timeout 後就會對當前持有 GIL 正在執行中的 thread 設定其 `gil_drop_request` 為 1，並再次進入待命週期。
  - 最先 timeout 的 thread 並不一定會取得 GIL，這取決於 OS 對於各個 thread 的優先級排序，而剛取得 GIL 的 thread 會至少運行一個週期後才會考慮釋出 GIL。
- 新的 GIL 機制可以有效的減少舊版本中過度頻繁的 context switch，但可能也會因此導致 CPU-bound 的 thread 持有 GIL 更長的時間，讓 IO-bound 的 thread 需要更長的等待。

#### Reference
- [New GIL](http://www.dabeaz.com/python/NewGIL.pdf)
