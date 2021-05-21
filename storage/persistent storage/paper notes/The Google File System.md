# The Google File System

###### tags: `fault tolerance`, ` scalability`, `data storage`, `clustered storage`

# 摘要
- 本文內容主要是講解他們如何實現一個適合 Data-Intensive Applications 的可擴展大型分散式檔案系統。
  - 這裡的 Data-Intensive Applications 指的是軟體中資料在各個面向上的複雜度，畢竟任何儲存服務如 Database 其底層仍舊是檔案的組成，而隨著情境的變化和流量的提升，在數量級、讀寫頻率、與結構改變上都有了新的挑戰。
- 同時透過在大量的評價設備上運行服務來實現局部容錯性。
- 最初的動機是因為透過觀察到目前軟體服務的工作量的性質，和先前做的假設明顯不同，因此需要重新檢視。
