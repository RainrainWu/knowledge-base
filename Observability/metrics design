# Metrics Design
- 具有特定意義的指標和視覺畫圖表可以快速地提供當前服務狀態的 global view。

## Different Types of Metrics
- 根據所需要的關鍵資訊和 data source 的不同，大致有以下幾個種類可以切入思考。

### Service Oriented
- 聚焦於服務和使用者間互動的介面與成果，可以作為判斷關鍵功能是否正常運作的縮影．比如：
  - Netflix 把影片串流開啟成功率作為服務健康程度的關鍵指標。
  - 電商平台的付款成功率。
  - 社交媒體平台的發文成功率。

### Operating System Oriented
- 判斷設備當前工作負荷與資源用量的指標，由於如果出了問題沒第一時間發現幾乎都會引發慘劇，所以各大 IaaS 平台都會提供這部分的關鍵指標，比如：
  - CPU usage, RAM usage
  - Disk IO, Network IO
  - Response latency
- 這部分的資訊可能需要 higher privileges 來獲取，可能導致安全性問題，同時不同 OS 的指令可能也不同，也許就交由 IaaS 平台最為合適。

### Programming Language Oriented
- 算是比較深入的監控，通常是需要深入鑽研效能問題或是記憶體開銷量時，會依據各個程式語言的 runtime 機制來決定獲取哪些關鍵資訊。
- 以 Go 語言 ver1.12 之後的版本而言，可以用 `runtime` 這個 package 來取得資訊，比如：
  - `GOMAXPROCS`、`GC Percentage` 等等靜態參數，所是某種意義上的 metadata，輔助判斷當前服務運作狀況是否合乎預期。
  - GC 發生的時機。
  - goroutine 數量。
