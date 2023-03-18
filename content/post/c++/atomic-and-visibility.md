---
title: "原子操作可见性"
date: 2023-03-18T15:57:59+08:00
toc: true

---

# 问题

当 thread1 执行完毕后，thread2 能立马读到 7 吗？

```C++
std::atomic<int> x{0};
// Thread 1
x.store(7, std::memory_order_relaxed);
// Thread 2
x.load(std::memory_order_relaxed);
```

当 thread1 和 thread2 都执行 f 后，x 的值一定是 200 吗？

```C++
std::atomic<int> x{0};
void f() {
    for (int i = 0; i < 100; i++) {
        x.fetch_add(1, std::memory_order_relaxed);
    }
}
// Thread 1
f();
// Thread 2
f();
```

# 分析

对于第一个问题，thread2 可能无法立马读到 7。对于第二个问题，保证 x 的值一定是 200。

原子操作保证不可分割，对原子类型的读写保证没有数据竞争。

可见性指一个线程写一个变量后，其他线程是否能够看到最新的值。

Relaxed memory order 能够保证所有线程看到对同一个变量的修改序是一致的。假如原子变量 x 初始值为 0， 变量先修改为 1，后修改为 2，那么所有线程能读到 x 的顺序只能是 0-1-2。

对于第一个问题，store 和 load 之间并没有保证 load 一定能读到最新的值，只能保证读到的值的顺序是符合修改序的，且所有线程一致。c++20  标准只是说需要在一个有限的时间能同步到其他线程。

> An implementation should ensure that the last value (in modification order) assigned by an atomic or synchronization operation will become visible to all other threads in a finite period of time.

但是对于  read-modify-write (RMW)，c++ 标志规定了 RMW 一定是读到最新的值，RMW 是更昂贵的操作。

> Atomic read-modify-write operations shall always read the last value (in the modification order) written before the write associated with the read-modify-write operation.

shared_ptr 内部的引用计数使用原子变量，由于增加计数是 RMW，所以保证读到最新的值，保证计数的准确性。

上面两个问题都是对一个原子变量的读写，都和内存序没有关系，内存序通过原子变量用来建立 "happens-before" 关系。 