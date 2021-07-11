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
```bash=
1 time duration: 0.0007641330000000002, 10 times duration: 0.006023733
3 times experiments: [0.0006617430000000028, 0.0005786179999999995, 0.0005808799999999989]
```
