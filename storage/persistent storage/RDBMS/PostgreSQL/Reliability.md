# Reliability
- 為了能確保資料庫的運作在斷電、網路異常等等極端環境中盡可能不厚影響，有足夠的可靠性確保其保存資料的品質是很重要的。

## Write-ahead Log (WAL)
- PostgreSQL 主要透過將所有 commit 的 transaction 都預先紀錄至永久儲存的 Log 後才開始對資料的改動操作的方式，只要儲存磁碟並未受到致命損壞就可以完整轉移所有曾經的資料。
- 由於 transaction 是對資料庫操作的最小單位，因此基於 transaction 紀錄的 WAL 也可以實現時間點備份還原 PITR (point-in-time recovery)。
- 某些對效能有高度要求的服務中，資料庫可能先完成 WAL 但還沒對資料完成更新改動前就回復成功，但此作法可能遭遇臨時意外而導致已回報成功 transaction 尚未執行完畢，因此對於需要依照 transaction 結果成功與否來判斷下一步操作的情境則無法適用。
- WAL 所記錄的 log file 可支援基於時間限制或檔案大小限制的檔案滾動，但仍要記得定期備份舊的資料以避免將 production 環境中的儲存空間耗盡。
- WAL 的備份並不是完全無懈可擊的，比如單一筆 transaction 意外導致資料錯誤就會連帶影像後續所有 transaction，因此定時製作整張資料表的 snapshot 備份仍是有必要的。
- 在 cluster 中多個 PostgreSQL instance 做 streamming replication 時也是基於 WAL 機制，不斷向 standby 節點傳送新的 transaction 紀錄來實現。
