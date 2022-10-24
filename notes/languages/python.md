# Python (CPython)
- [Python (CPython)](#python-cpython)
  - [Scheduling](#scheduling)
    - [GIL (Global Interpreter Lock)](#gil-global-interpreter-lock)
  - [Memory Management](#memory-management)
    - [PyMalloc](#pymalloc)
    - [Garbage Collection](#garbage-collection)
      - [References Counting](#references-counting)
      - [Generational Garbage Collection](#generational-garbage-collection)
    - [Tuning Tips](#tuning-tips)
      - [Weak Reference](#weak-reference)
      - [Manual Garbage Collection](#manual-garbage-collection)
      - [String Interning](#string-interning)
  - [Profiling](#profiling)
    - [Time](#time)
    - [Memory](#memory)
  - [Object Oriented Programming](#object-oriented-programming)
    - [Metaclass](#metaclass)

## Scheduling

### GIL (Global Interpreter Lock)
- GIL ensure only one thread is executing byte code and manipulating the underlying data object.
- The reason why the limitation is required is the memory management behavior is not thread-safe.
- GIL may degrade the advantages while utilizing multi-core device, but not all kinds of tasks require GIL for execution (e.g. network I/O, file processing, ...).

**References**
- [Global Interpreter Lock](https://wiki.python.org/moin/GlobalInterpreterLock)
- [New GIL](http://www.dabeaz.com/python/NewGIL.pdf)

## Memory Management

### PyMalloc
- All of the objects which smaller than 512 bytes will be managed by PyMalloc, other larger objects were taken over by standard C allocator.

**References**
- [Memory management in Python](https://rushter.com/blog/python-memory-managment/)

### Garbage Collection

**References**
- [Garbage Collection in Python](https://rushter.com/blog/python-garbage-collector/)
- [Python Garbage Collection: What It Is and How It Works](https://stackify.com/python-garbage-collection/)

#### References Counting
Each strong reference against an object increase it's ref count number, it will be recycled once the number becomes zero.

#### Generational Garbage Collection
To mitigate with circular reference, Python also introduce generational gc as an alternative.

There are 3 generations with a threshold of total object amount, gc will be triggered if any of threshold was exceeded.

### Tuning Tips

#### Weak Reference
To workaround with the memory consumption against large object or cached data, weak ref will not contribute to the reference counts and the target object may be recycled by gc automatically.

**References**
- [weakref – Garbage-collectable references to objects](http://pymotw.com/2/weakref/)

#### Manual Garbage Collection
Only generational garbage collection can be tuned manually, several of the settings can be patched via standard `gc` module.

**References**
- [gc – Garbage Collector](http://pymotw.com/2/gc/)

#### String Interning
Since Python 3, the `str` type uses Unicode representation. Unicode strings can take up to 4 bytes per character depending on the encoding, which sometimes can be expensive from a memory perspective.

- This approach interned strings act as singletons, that is, if you have two identical strings that are interned, there is only one copy of them in the memory.
- Constant strings that are created during runtime can also be interned if their length does not exceed 20 characters.
- We can also specify the interning behavior manually.
    ```python
    from sys import intern

    interned_str = intern("STRING_TO_BE_INTERNED")
    ```

**References**
- [How Python saves memory when storing strings](https://rushter.com/blog/python-strings-and-memory/)

## Profiling

### Time

**timeit**
```python
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

**line_profile**
```bash
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

### Memory

**memory_profiler**
```bash
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

**memray**
- [GitHub Repository](https://github.com/bloomberg/memray)

## Object Oriented Programming

### Metaclass

Metaclass is the class for class, that is, we can use metaclass to instantiate class instance even dynamically at the runtime.

**References**
- [Python Metaclasses](https://realpython.com/python-metaclasses/)
