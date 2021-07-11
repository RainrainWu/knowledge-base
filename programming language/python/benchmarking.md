# Benchmarking
- 對 python code 測效能的工具百寶箱。

## Time
- 以檢視執行時間為主的效能測試。

### timeit
- 主要用於整個函數的效能測試。
  - `number` 控制單次測驗週期的執行次數。
  - `repeat` 控制實驗重複次數。

#### example
- code
```python3
import timeit

def func():
    buff = []
    for i in range(10000):
        buff.append(1)

one_t = timeit.Timer(func).timeit(number = 1)
ten_t = timeit.Timer(func).timeit(number = 10)
print(f"1 time duration: {one_t}, 10 times duration: {ten_t}")

three_r = timeit.Timer(func).repeat(repeat=3, number=1)
print(f"3 times experiments: {three_r}")

```
- output
```
1 time duration: 0.0007641330000000002, 10 times duration: 0.006023733
3 times experiments: [0.0006617430000000028, 0.0005786179999999995, 0.0005808799999999989]
```

### line_profile
- 可以更細緻的對每一行程式計算算執行時間，適合用於使用大量高抽象封裝函式的情境，可能效能平靜就在其中某個 funciton call。
  - Hit 是該行被執行的次數。
  - Time 是該行執行的總時間。
  - Per Hit 指的是平均每次該行被執行的時間。
  - % Time 指的是時間佔比，可以優先考慮改善佔比最高的指令。
- 第三方套件，需要先安裝 [line_profiler](https://pypi.org/project/line-profiler/)

#### example
- code
```python3
def foo():
    buff = []
    for i in range(10000):
        buff.append(i)

@profile
def bar():
    print("call funciton foo")
    foo()
    print("function foo complete")

bar()
```
- commands
```
$ poetry run kernprof -l test.py
call funciton foo
function foo complete
Wrote profile results to test.py.lprof

$ poetry run python -m line_profiler test.py.lprof
Timer unit: 1e-06 s

Total time: 0.002729 s
File: test.py
Function: bar at line 6

Line #      Hits         Time  Per Hit   % Time  Line Contents
==============================================================
     6                                           @profile
     7                                           def bar():
     8         1         33.0     33.0      1.2      print("call funciton foo")
     9         1       2685.0   2685.0     98.4      foo()
    10         1         11.0     11.0      0.4      print("function foo complete")
```

## Memory
- 以檢視記憶體佔用為主的效能測試。

### Memory_profiler
- 可以分析記憶體佔用的工具，同樣是可針對每一行指令做追蹤。
  - Mem usage 是總累積用量。
  - Increment 是該行指令所增加的用量。
  - Occurences 是該行指令被執行的次數。
- 是第三方套件，需要先安裝 [memory-profiler](https://pypi.org/project/memory-profiler/)

#### example
- code
```python3
def foo():
    buff = []
    for i in range(10000):
        buff.append(i)

@profile
def bar():
    print("call funciton foo")
    foo()
    print("function foo complete")

bar()
```
- commands
```
$ poetry run python -m memory_profiler test.py
call funciton foo
function foo complete
Filename: test.py

Line #    Mem usage    Increment  Occurences   Line Contents
============================================================
     6   37.609 MiB   37.609 MiB           1   @profile
     7                                         def bar():
     8   37.617 MiB    0.008 MiB           1       print("call funciton foo")
     9   37.711 MiB    0.094 MiB           1       foo()
    10   37.711 MiB    0.000 MiB           1       print("function foo complete")

```
