# Load Balancing
- 為了分攤過大的工作量，同時也避免 single point of failure，後端會由多台 app server 組成，透過 load balancing 的技術去分散流量。

## Layer 4 v.s. Layer 7
- 分別是傳輸層和應用層的分流
  - Layer 4 僅以 IP address 和 port 進行網路封包導流，並不會解析請求內容，因此無法做更細節的優化，但也相對高性能一些。
  - Layer 7 會針對請求的協議解析內容，可以透過 url、cookie 等等資訊進一步選擇導向的目標 app server，但也因此效率會低一些。

## Load Balancing Algorithms
- 針對不同的業務性質可以選用合適的演算法來作分流。
  - Round robin 的演算法會定期切換目標 app server，在多數情況都不會產生極端糟糕的情況，也是多數 load balancer 的預設演算法。
  - Less connect 會選用當前 connection 數最少的 load balancer 導流，適合用於如 websocket 之類長連結的情境。
  - Source 會針對 IP address 的 hash 值來決定導向哪個 app server，可以確保同個 user 盡可能都連接到同個 app server，但也同時喪失了些平衡工作量的靈活性。

## Common Issues

### Sticky Session
- 若 app server 中有使用 session 之類的技術，便會使得 server 具有 stateful 的特性，需要每次都連接同個 app server 才會有操作的連貫性。
- 這時的解決方法要嘛就是把 session 放在一個外部的儲存空間讓所有 server 共用，或者針對 IP address 來建立 routing table 讓同個 user 能連到同個 server。

### Long Connection
- 許多資料傳輸量較大的服務會採用 websocket 的技術來避免頻繁 TCP connection 的連線成本，但也因此失去了分配同一連線內工作量的靈活度。
- 這個情境的解法主要可以靠 reconnect 來實現，不過 reconnect 同時也有影響使用體驗甚至連線失敗的風險，因此需要慎選時機。
- 若是初期採用或是 user story 難以掌握的話就先定期 reconnect，若對使用者的體驗有深入研究的話可以刻意挑選合適的時機，比如：
  - 開始串流新的直播前
  - 下載大型檔案前

### High Availability (HA)
- 雖說多個 app server 可以有效避免 single point of failure，但 load balancer 若只有一個仍然無法實現完整的 HA。
- 一個理想的做法是對外提供 elastic IP 做接口，並且同時維護多個 load balancer 並加以監控，一旦當前負責導流的節點有問題就立刻將 elastic IP assign 給另個節點。
- [參考圖片來源](https://www.digitalocean.com/community/tutorials/an-introduction-to-haproxy-and-load-balancing-concepts)
  ![](https://assets.digitalocean.com/articles/high_availability/ha-diagram-animated.gif)

## References
- [The Advanced Challenge of Load Balancing](https://medium.com/geekculture/the-advanced-challenge-of-load-balancing-6f6ef5f36ec4)
