---
title: c++11之std::atomic_flag与std::atomic
date: 2022-09-19 15:33:42
tags: ["c++","多线程","atomic"]
---


<!-- more -->
# 原子操作
**原子操作**(atomic operation)是一个不可分割的操作。从系统中的任何一个线程中，都无法观察到一个完成到一半的这种操作，它要么做完了，要么就没做完。如果读取对象值的载人操作是原子的(atomic)，并且所有对该对象的修改也都是原子的，那么这个载人操作所获取到的要么是对象的初始值，要么是被某个修改者存储后的值。
标准原子类型可以在 \<atomic>

# std::atomic_flag
- std::atomic_flag是一个bool类型的原子变量，它拥有两个状态set和clear，对应flag的true和false
- std::atomic_flag使用前必须被ATOMIC_FLAG_INIT进行初始化，此时的flag为clear状态，相当于静态初始化

主要提供两个原子化操作:
1. test_and_set()：检查当前flag是否被设置，如果已设置直接返回true，若没有设置则将flag设置为true并且返回false
2. clear()：用来清除flag标志，即flag=false

看下面一段代码：
```c++
#include <iostream>
#include <vector>
#include <thread>
#include <atomic>

std::atomic_flag lock = ATOMIC_FLAG_INIT;

void print_fnc(int num) {
    //while (lock.test_and_set());注释掉同步操作
    std::cout << "thread " << num << " is working !!" << std::endl;
    lock.clear();
}
int main()
{
    std::vector<std::thread> ths;

    for (int i = 0; i < 10; ++i) {
        ths.emplace_back(std::thread(print_fnc, i));
    }
    for (auto& th : ths) {
        th.join();
    }
    return 0;
}
```
在本地得到的某一次输出为：
```shell
thread thread 10 is working !! is working !!thread 8 is working !!thread 3 is working !!
thread 2 is working !!

thread thread 6 is working !!thread 5 is working !!thread


7 is working !!
4 is working !!
thread 9 is working !!
```
这里我们在print_fnc中打印了一些信息，然后创建了10个线程，每个线程的入参都不一样，从输出我们可以看到这10个线程是异步进行的，所以出现了某一行有很多打印信息，某一行只有一个空行，当我们解注释掉第9行代码后得到的某个输出为：
```shell
thread 0 is working !!
thread 4 is working !!
thread 3 is working !!
thread 7 is working !!
thread 6 is working !!
thread 8 is working !!
thread 9 is working !!
thread 5 is working !!
thread 1 is working !!
thread 2 is working !!
```
可以看到虽然执行顺序不是按照thread的创建顺序来，但是等到某个线程clear了lock后才开始执行下一个线程，因此std::atomic_flag可以用来做多线程之间的同步操作，类似于Linux中的信号量。使用atomic_flag可实现mutex

std::atomic_flag不支持拷贝和赋值等操作，因为这两个操作设计到两个对象，必定不是原子操作

std::atomic_flag类型不提供is_lock_free。该类型是一个简单的布尔标志，并且在这种类型上的操作都是无锁的，但是std::atomic_flag的可操作性不强，导致其应用具有局限性，不如std::atomic\<bool>


# std::atomic
std::atomic<>是一个类模板，除了可以特化基本类型外还可以用于创建一个用户自定义类型的原子变种，由于它是一个泛型类模板，操作只限为 load(), store(),（和与用户定义类型间的相互赋值）, exchange(), compare_exchange_
weak() 和 compare_exchange_strong()
在原子类型上的每一个操作均具有一个可选的内存顺序参数，它可以用来指定所需的内存顺序语义，三种运算类型的顺序为：
- **存储**(store) 操作：可以包括 memory_order_relaxed, memory_order_release 或 memory_order_seq_cst 顺序
- **载入**(load) 操作：可以包括 memory_order_relaxed, memory_order_consume, memory_order_acquire 或 memory_order_seq_cst 顺序
- **读-修改-写**(read-modify-write) 操作：可以包括 memory_order_relaxed, memory_order_consume, memory_order_acquire, memory_order_release, memory_order_acq_rel 或 memory_order_seq_cst 顺序
所有操作的默认顺序为 memory_order_seq_cst