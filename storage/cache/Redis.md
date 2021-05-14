# Redis
- Redis 是一個以 key-value pair 為儲存結構，支援的資料型別也包含 array, set, hash 等等多元資料型別。
- 將資料保存在記憶體中以提昇操作效率。
- 雖然記憶體中的資料是揮發性的，但 Redis 也可以支援定時 snapshot 寫入持久化儲存的備份功能。

## 潛在問題

### 空間限制
- 資料庫所儲存的資料量肯定會比 cache 儲存的資料量大的多的，而若要受惠於 cache 的機制通常會先向資料庫取得資料後存入 cache 再回覆給使用者，久而久之 cache 的空間便會有耗盡的問題。
- Redis 針對這問題有兩個方法可以解決一個是設置每對 key-value pair 的過期時間，也就是 TTL (time-to-live)，另個是設定強制淘汰機制 eviction policy。

#### TTL (time-to-live)
- Redis 中可以設置每對 key-value pair 的過期時間，若沒特別指定的話預設就是無限期保存。
- 但為了效能和可用性考量所以不會週期性的確認每一個 key-value pair 的效期，而是隨機抽樣其中幾個，若有發現效期已經過的就會移除。
- 同時在使用者嘗試存取一對 key-value pair 時，也會先檢查其效期是否過期，以避免娶到過期的數值。
- 但以上兩種方法並無法完全避免儲存空間用盡的狀況，因此仍需要強制淘汰機制。

#### Eviction Policy
- 當所剩空間不足以存下新資料時，就需要選擇一些 key-value pair 來強行刪除以騰出足夠空間，而如何選擇就是透過這個 policy 來實現。
- 目前提供針對未設置 TTL 期限之資料的 random, lru，和有設置 TTL 期限之資料的 random, lru 等等策略，但若選擇針對有設置 TTL 的資料淘汰卻沒有為任何一筆資料設定 TTL 的話就會引發錯誤。

### Cache Penetration
- 具體情況就是瞬間湧入大量的請求取存取 cache 中沒有留存的資料，導致所有請求都直接以資料庫為對象，cache server 形同虛設。
- 這種情況多半是由惡意攻擊引起，解決方法有許多彈性，比如：
  - 維護一定數量的 lock 來限制同時間向資料庫存取的壓力，只有取得 lock 的才能主動向資料庫拿資料。
  - 把向資料庫存取資料的任務改為非同步執行，可針對請求直接返回 server busy，但背景中的 async job 仍會更新 cache 中的值。
  - 透過一些 middleware 先行擋住疑似腳本發起的請求。

### Cache Avalanche
- 具體情況是短時間有大量的資料到期，導致大量的請求也會直接以資料庫為存取對象，cache server 形同虛設。
- 此類問題類似於分散式系統的 thundering herds，只需要在設定 TTL 時乘上一個 random number 上每筆紀錄的到期時間有所差異。

### 併發寫入
- 當多個 task 併發執行並針對同一筆紀錄做更新時，可能爭搶 write lock。
- 若是單一 redis server 的情況下可以用 transaction 強行持有執行權限直到所有 command 依序執行完成，期間不會受到其他操作干擾，也就不會有搶 lock 的問題。
- 但在多節點的 cluster 環境中 transaction 並無法規範分布在不同節點上的資料，因此有點雞肋不一定能達成效果。
- 若不要求執行順序的話，就在 application tier 設計一個鎖來控制，否則就是戴上時間戳後放入 queue 來依序執行。
