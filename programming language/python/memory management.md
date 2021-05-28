# 記憶體管理
- Python 是一門支援 automatic memory management 的語言，除了可以節省心力提升開發速度外，也可以避免忘記釋出記憶體空間或是過早釋出的問題。
- 這部分內容是以較為廣泛被採用的 CPython implementation 為主。

## 垃圾回收
- 回收已經不需使用的記憶體空間是其中一個重要的工作，Python 同樣有自動垃圾回收的機制，主要有兩個不同的演算法：

### Reference Counting
- 是 CPython 中主要的垃圾回收機制。
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
- 若希望嚴格控制 GC 的時機避免與工作量高峰相撞，可以使用 `gc.disable()` 和 `gc.collect()` 來控制。

## Reference
- [Garbage collection in Python: things you need to know](https://rushter.com/blog/python-garbage-collector/)
