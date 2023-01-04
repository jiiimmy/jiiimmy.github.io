---
title: c++11之std::atomic_flag与std::atomic
date: 2022-09-19 15:33:42
tags: ["c++","多线程","atomic"]
---

这篇博客用来总结 std::atomic 相关用法
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

std::atomic_flag类型不提供is_lock_free()。该类型是一个简单的布尔标志，并且在这种类型上的操作都是无锁的，但是std::atomic_flag的可操作性不强，导致其应用具有局限性，不如std::atomic\<bool>


# std::atomic
std::atomic<>是一个类模板，除了可以特化基本类型外还可以用于创建一个用户自定义类型的原子变种，由于它是一个泛型类模板，操作只限为 load(), store(),（和与用户定义类型间的相互赋值）, exchange(), compare_exchange_
weak() 和 compare_exchange_strong()
在原子类型上的每一个操作均具有一个可选的内存顺序参数，它可以用来指定所需的内存顺序语义，三种运算类型的顺序为：
- **存储**(store) 操作：可以包括 memory_order_relaxed, memory_order_release 或 memory_order_seq_cst 顺序
- **载入**(load) 操作：可以包括 memory_order_relaxed, memory_order_consume, memory_order_acquire 或 memory_order_seq_cst 顺序
- **读-修改-写**(read-modify-write) 操作：可以包括 memory_order_relaxed, memory_order_consume, memory_order_acquire, memory_order_release, memory_order_acq_rel 或 memory_order_seq_cst 顺序
所有操作的默认顺序为 memory_order_seq_cst

## std::atomic<bool>
std::atomic\<bool> 是最基本的原子整数类型 ，这是一个比 std::atomic_flag 功能更全的布尔标志，虽然它仍不可进行拷贝构造和拷贝赋值，但是可以从一个非原子的 bool来构造，也可以用一个非原子的 bool 值来对其赋值
```c++
std::atomic<bool> b(true);
b = false;
```
从非原子的 bool 进行赋值操作并非向其赋值的对象返回一个引用，它返回的是具有所赋值的 bool。这对于原子类型是另一种常见的模式，它们所支持的赋值操作符返回的是**值**（属于相应的非原子类型）而**不是引用**。
如果返回的是原子变量的引用，所有依赖于赋值结果的代码将显式地载人该值，可能会获取被另一线程修改的结果。通过以非原子值的形式返回赋值结果，可以避免这种额外的载人。
看下面一段代码：
```c++
#include <atomic>
#include <iostream>
int main() {
  std::atomic<bool> b;
  bool x = b.load(std::memory_order_acquire);
  std::cout << "x = " <<x <<std::endl;
  b.store(true);
  x = b.exchange(false, std::memory_order_acq_rel);
  std::cout << "x = " <<x <<std::endl;
  std::cout << "b = " << b.load() <<std::endl;
  
  return 0;
}
```
程序的输出为：
```shell
x = 0
x = 1
b = 0
```
从输出我们可以看到 std::atomic\<bool> 的默认构造是赋值false， store() 和 exchange() 的区别是 store() 只负责写，不管原来是什么， exchange()会读原来的数据，然后修改并写成新数据，最后返回原数据

exchange()并非 std::atomic\<bool>唯一支持的读-修改-写操作，它还引入了**比较/交换**的操作，用于在当前值与期望值相等时，存储新的值。对应的成员函数为 compare_exchange_weak() 和 compare_exchange_strong()。它比较原子变量值和所提供的期望值，如果两者**相等**，则**存储提供的期望值**。如果两者不等，则**期望值更新为原子变量的实际值**。比较/交换函数的返回值类型为bool，如果执行了存储则返回true，否则为false

<!-- 对于 compare_exchange_weak() -->

## std::atomic<T*>
对于某种类型 T 的指针的原子形式是 std::atomic\<T*>，其接口基本上与 std::atomic\<bool>是相同的，只不过它对相应的指针类型的值进行操作而非 bool 值。它也不能拷贝构造和拷贝赋值，虽然它可以从合适的指针值中构造和赋值。和必须的 is_lock_free()成员函数一样， std:: atomic\<T*>也有 load()、 store()、 exchange()、compare_exchange_weak()和 compare_exchange_strong()成员函数，具有和std::atomic\<bool>>相似的语义，也是接受和返回 T* 而不是 bool 。

std::atomic\<T*>提供的新操作是指针算数运算，这种基本的操作是由 fetch_add()和 fetch_sub()成员函数提供的，可以在所存储的地址上进行原子加法和减法，+=, -= 带 ++ 和 -- 前缀与后缀的自增自减等运算符，都提供了方便的封装 。
fetch_add() 和 fetch_sub() 有细微的区别，他们返回的**都是值而不是引用**，对于 x.fetch_add(3) 会将 x 更新为指向第四个值，但是返回一个指向数组中第一个值的指针。而 x.fetch_sub(2) 会将 x 更新为前两个值，返回的是 x 实际指针的值，看下面一段代码：
```c++
#include <atomic>
#include <iostream>
int main() {
  class Foo {};
  Foo some_array[5];
  std::atomic<Foo *> p(some_array);
  std::cout << "a    ptr = " << some_array << std::endl;
  Foo *x = p.fetch_add(2);
  std::cout << "x    ptr = " << x << std::endl;
  std::cout << "p    ptr = " << p.load() << std::endl;
  std::cout << "a[2] ptr = " << &(some_array[2]) << std::endl;
  std::cout << "--------" << std::endl;
  x = (p -= 1);
  std::cout << "x    ptr = " << x << std::endl;
  std::cout << "p    ptr = " << p.load() << std::endl;
  std::cout << "a[1] ptr = " << &(some_array[1]) << std::endl;
  
  return 0;
}
```
程序的输出为：
```shell
a    ptr = 0x7ffcd0933de3
x    ptr = 0x7ffcd0933de3
p    ptr = 0x7ffcd0933de5
a[2] ptr = 0x7ffcd0933de5
--------
x    ptr = 0x7ffcd0933de4
p    ptr = 0x7ffcd0933de4
a[1] ptr = 0x7ffcd0933de4
```
从上面代码的输出能够看出 fetch_add()和 fetch_sub() 的细微差异， 同时这两个函数也允许内存顺序语义作为一个额外的函数调用参数

## std::atomic<>初级类模板
初级模板的存在允许用户创建一个用户定义的类型的原子变种。但是该类型必须满足一定的准则。为了对用户定义类型 UDT 使用 std::atomic\<UDT>，这种类型必须有一个平凡的(trivial) 拷贝赋值运算符。这意味着该类型**不得拥有任何虚函数或虚基类**，并且**必须使用编译器生成的拷贝赋值运算符**。不仅如此，一个用户定义类型的每个基类和非静态数据成员也都必须有一个平凡的拷贝赋值运算符。这实质上允许编译器将 memcpy() 或一个等价的操作用于赋值操作，因为没有用户编写的代码要运行。
最后，该类型必须是按位相等可比较的。即必须能够使用 memcmp() 比较实例是否相等。为了使比较／交换操作能够工作，这个保证是必需的。

## 原子类型的可用操作
| 操作                    | atomic_flag | atomic\<bool\> | atomic\<T*\> | atomic\<integral-type\> | atomic\<other-type\> |
| ----------------------- | ----------- | -------------- | ------------ | ----------------------- | -------------------- |
| test_and_set            | √           |                |              |                         |                      |
| clear                   | √           |                |              |                         |                      |
| is_lock_free            |             | √              | √            | √                       | √                    |
| load                    |             | √              | √            | √                       | √                    |
| store                   |             | √              | √            | √                       | √                    |
| exchange                |             | √              | √            | √                       | √                    |
| compare_exchange_weak   |             | √              | √            | √                       | √                    |
| compare_exchange_strong |             | √              | √            | √                       | √                    |
| fetch_add, +=           |             |                | √            | √                       |                      |
| fetch_sub, -=           |             |                | √            | √                       |                      |
| fetch_or,   \|=         |             |                |              | √                       |                      |
| fetch_and, &=           |             |                |              | √                       |                      |
| fetch_xor, ^=           |             |                |              | √                       |                      |
| ++, --                  |             |                |              | √                       |                      |

