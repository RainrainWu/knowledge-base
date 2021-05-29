# 記憶體管理
- Python 是一門支援 automatic memory management 的語言，除了可以節省心力提升開發速度外，也可以避免忘記釋出記憶體空間或是過早釋出的問題。
- 這部分內容是以較為廣泛被採用的 CPython implementation 為主。

## Memory Allocation
- Python 在運行時會有非常大量小型物件，在 CPython 的實現中所有小於 512 bytes 的物件都會由特製的 PyMalloc 管理器來管理，更大型的物件則是直接由 standard C allocator 來配置。

### PyMalloc
- PyMalloc 主要將記憶體資源分為三個階層管理，由小到大分別為 block, pool, arena。

#### Block
- 每個 Block 都是預先切分好的記憶體區塊，每個 Block 都只能儲存一個 Python 中的 object。
- 總共有 64 種大小不同的 Block，從 8 bytes 至 512 bytes 體積都是 8 的倍數，當需要儲存 object 便會取用最合適的 block 來儲存。

#### Pool
- pool 內會儲存大量大小相同的 Block，彼此之間用 double-linked list 結構連結，通常一個 Pool 的大小就是一個 memory page 的大小。
- 每個 pool 都會有一個 \*freeblock 的指標持續維護一個目前閒置中可用 free block 的 singly-linked list，方便快速的找到可用的 block。
  - 當一個 Block 沒有儲存任何 object 時，他會儲存同一個 pool 內下個可用的 block 的 address，這也可以省掉連續取用同 pool 中 block 的搜索時間。
  - 在 pool 初始化時只會將前兩個 block 放入 \*freeblock 中，隨著不斷取用 block 她都可以很快的透過下個可用 block 的 address 遞補進來這個 list 中。
  - 而一但有個 block 不在被使用而被釋出，他便會 insert 到 \*freeblock 的最前端，整個管理策略慧卿相盡可能不用到最後一個 block 為目標。
- pool 的使用狀態分為三種：empty, used, full。
- 為了快速找到各個層級的 pool，CPython 也會維護一張 usedpools 的清單，內部包含指向各個層級 pools 的指針，而同個層級的 pool 之間彼此也以 souble-linked list 來連結。

#### Arena
- 每個 arena 都是在 heap 上一個 256KB 的 chunk，負責配置記憶體空間給 block 和 pool 使用，並存有當前 arena 內可用 pool 的指針及 pool 總數等等 metadata。
- arena 之間彼此也透過 souble-linked list 連結方便管理。

### Deallocation
- PyMalloc 的機制其實不會立即把沒在使用的記憶體空間還給 OS。
  - 因為 OS 配發記憶體空間的對象是 arena，再由 arena 供給給底下的 pool 和 block 使用。
  - 反過來說，PyMalloc 也只會在一個 arena 內所有的 pool 都清空時才會將整個 arena 的空間歸還。
- 因此若有個 Python process 執行很長一段時間的話，其中表面上佔用的空間中可能有不少是閒置的，可以透過 `sys._debugmallocstats()` 來檢視細節。

### Reference
- [Memory management in Python](https://rushter.com/blog/python-memory-managment/)

## Gargabe Collection
- 回收已經不需使用的記憶體空間是其中一個重要的工作，Python 同樣有自動垃圾回收的機制，主要有兩個不同的演算法：

### Reference Counting
- 是 CPython 中主要的垃圾回收機制，開發者無法決定啟用或停用。
- Python 中的每個物件都有其生命週期，而在程式碼中使用的 variable（或稱作 identifier）都是對物件的引用，每多被引用一次該物件的 reference count 就會增加一（比如 assignment operator, argument passing），而取消引用後就會減一。
  - 個別 variable 的 reference count 可用 `sys.getrefcount` 來查看。
  - 在 global scope 所宣告的變數通常會一直留存著，可以使用 `global()` 函數來查看，這也表示對 global variable 所 reference 的物件其 reference count 都不會降至 0，因此儘可能別大量使用 global variable。
  - 在 local scope 比如 class 或 funciton 內所宣告的變數，只要 Python interpreter 離開了該 block 便會移除 scope 內所有變數對物件的引用。
  - 換句話說ㄝ當 Python interpreter 正執行到一個 function 中時，他會假設該 function scope 內的所有 variables 都在使用中而不會移除他們對物件的 reference，因此勁量別讓單個函數過於繁重冗長以避免長時間無法釋放閒置中的 reference。
  - 若要立刻強行移除一個變數對於背後物件的 reference，可以使用 `del`。
- Python 會挑選合適的時機點回收掉 reference count 降至 0 的物件。
- 這做法的好處一旦物件可被回收時就能立刻知道，但仍有些缺陷如 Cyclic reference 會無法進行處理、對 counting 需增加 Lock（這也是其中一個 Python 需要 GIL 的原因）。
- 在 Python 1.5 後新增了 cyclic reference 的偵測機制，主要是針對 list, set, tuple, dict 等等內部可再複合引用其他變數的 container variables，可以透過 `gc` 這個 module 來檢視具體情況。

### Generational Garbage Collection
- 相對於 reference counting 持續 real time 運作，generational GC 是採取週期性觸發的方式來執行。
- 其中一共有三個 generation，所有物件剛被創建時都在第一個世代，每過一個週期就會往下至下個世代，而每個層級都有其容量限制（預設是 700, 10, 10），一旦達到限制就會觸發最回收機制並同時回收比自己更年輕的世代。
  - 不過最古老的第三層級有額外的觸發條件。
- 通常較低層級的物件會頻繁的需要垃圾回收，因為有大量的小型物件只會在短時間為了特定功能使用。

## Performance Tips
### Weak Reference
- 若透過該變數存取的物件並非至關重要，可以使用 `weakref` 這個 module，它可以讓變數在被 assign 時不會直接增加背後物件的 reference count，若該物件採存取時已經被刪除則會回傳 None。

### Manual GC
- 若希望嚴格控制 GC 的時機避免與工作量高峰相撞，可以使用 `gc.disable()` 和 `gc.collect()` 來控制，只能控制 generational garbage collection 的機制。

### Reference
- [Garbage collection in Python: things you need to know](https://rushter.com/blog/python-garbage-collector/)
