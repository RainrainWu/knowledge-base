# Reliability
- 為了能確保資料庫的運作在斷電、網路異常等等極端環境中盡可能不厚影響，有足夠的可靠性確保其保存資料的品質是很重要的。

## Write-ahead Log (WAL)
- PostgreSQL 主要透過將所有 commit 的 transaction 都預先紀錄至永久儲存的 Log 後才開始對資料的改動操作的方式，只要儲存磁碟並未受到致命損壞就可以完整轉移所有曾經的資料。
- 某些對效能有高度要求的服務中，資料庫可能先完成 WAL 但還沒對資料完成更新改動前就回復成功，但此作法可能遭遇臨時意外而導致已回報成功 transaction 尚未執行完畢，因此對於需要依照 transaction 結果成功與否來判斷下一步操作的情境則無法適用。
- WAL 所記錄的 log file 可支援檔案滾動，再啟動 database 時透過 `–wal-segsize` 參數設定檔案大小限制。