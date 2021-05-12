# RDBMS 對 ACID 的要求
- 在對資料庫執行 transaction 操作時，為了確保資料在執行出錯、網路問題或斷電等等意外情境下的合法性，而在幾個特性和行為上有所要求。
- 但每個資料庫對 ACID 的定義不盡相同，就算官方文件宣稱自己滿足 ACID 原則也還是要看一下具體的行為描述。

## Atomic
- 一筆 transaction 中可包含多個 SQL command，但同個 transaction 中的 SQL command 只能全部都依序執行完畢，或者全部都不執行。
- 遭遇斷電之類的意外時，未被 commit 的 transaction 全數 rollback，需修復資料時資料庫必須以 transaction 為基本單位逐一修復。
- 在幾乎絕大多數的商業系統中，多個欄位的 atomic 性質都是必須的，比如轉帳時會牽扯到兩個帳戶的金額。

## Consistency
- 資料庫必須從一個正確的狀態，轉移到下一個正確的狀態，比如：
  - 使用者付錢買票，即使執行出錯也必須停留在「付錢且買到票」或是「沒付錢也沒買到票」的情況，不能是「付錢卻沒有票」或是「沒付錢卻拿到票」的情況。

## Isolation
- 不同 transaction 執行時不應該相互影響，也只能存取發起 transaction 時能夠存取的資料。
- Isolation 有多個層級可調整，理論上越嚴謹的隔離會犧牲越多效能，依據情況選擇合適的層級。
- 資料庫的隔離層級中最嚴格的就是 Serialize ，完全序列化確保每一筆 Transaction 逐一執行，而其他三個層級則是依據是否能妥善防止 transaction 間可能引發問題的相互干擾行為來劃分的。

## Durability
- 一筆 transaction 一旦被 commit 後，除非儲存空間受損否則將永遠不會消失。
