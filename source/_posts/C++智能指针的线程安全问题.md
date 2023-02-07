---
title: C++智能指针的线程安全问题
date: 2023-01-29 09:57:54
tags:
---


本篇博客介绍 C++ 智能指针的线程安全问题
<!-- more -->

先说结论，**智能指针都是非线程安全的**，以 shared_ptr 为例：

shared_ptr 的引用计数本身是安全且无锁的，但对象的读写则不是，因为 **shared_ptr 有两个数据成员，读写操作不能原子化**
- 一个 shared_ptr 对象实体可被多个线程同时读取
- 两个 shared_ptr 对象实体可以被两个线程同时写入，“析构”算写操作
- 如果要从多个线程读写同一个 shared_ptr 对象，那么需要加锁
请注意，以上是 shared_ptr **对象本身的线程安全级别**，不是它管理的对象的线程安全级别

# shared_ptr 的数据结构
shared_ptr 是引用计数型（reference counting）智能指针，采用在堆（heap）上放个计数值（count）的办法来实现引用技术。具体来说，shared_ptr<Foo> 包含两个成员，一个是指向 Foo 的指针 ptr，另一个是 ref_count 指针（其类型不一定是原始指针，有可能是 class 类型，但不影响这里的讨论），指向堆上的 ref_count 对象。ref_count 对象有多个成员，具体的数据结构如图所示，其中 deleter 和 allocator 是可选的
![shared_ptr 的数据结构](http://www.cppblog.com/images/cppblog_com/Solstice/WindowsLiveWriter/shared_ptr_1398A/sp0_bb14fcc3-adf1-417f-a5d2-439c8ab30a86.png)

简化来看，这里只考虑对象本身和引用计数：
![简化后的shared_ptr](http://www.cppblog.com/images/cppblog_com/Solstice/WindowsLiveWriter/shared_ptr_1398A/sp1_3.png)
上面的数据结构是我们执行一次 `shared_ptr<Foo> x(new Foo)`所对应的数据结构
如果再执行一次 `shared_ptr<Foo> y = x`，需要有2个步骤才能完成
1. 复制 ptr 指针 y.ptr = x.ptr
2. 复制 ref_count 指针，导致引用计数+1

这两个步骤的先后顺序与实现有关，基本都是先1后2
既然 y=x 有两个步骤，如果没有 mutex 保护，那么在多线程中就有 race condition

# 多线程无保护读写 shared_ptr 可能出现的 race condition
一个简单的场景，有三个 shared_ptr\<Foo>对象 x,g,n ：
* std::shared_ptr\<Foo> g(new Foo);//线程之间共享的 shared_ptr
* std::shared_ptr\<Foo> x;//读线程的局部变量
* std::shared_ptr\<Foo> n(new Foo);//写线程的局部变量

具体代码 test.cpp 如下：

```c++
#include <memory>
#include <thread>

class Foo {
public:
  Foo() : a(0) {}
  void get_val() {}

private:
  int a;
};

void read_thread(std::shared_ptr<Foo> &g) {
  std::shared_ptr<Foo> x;
  x = g;// read g
  x->get_val();// 当 x为空悬指针时，调用函数会 core dump
}

void write_thread(std::shared_ptr<Foo> &g) {
  std::shared_ptr<Foo> n(new Foo);
  g = n;// write g
}

int main() {

  for (int i = 0; i < 1000; ++i) {
    std::shared_ptr<Foo> g(new Foo);
    std::thread t1(read_thread, std::ref(g));
    std::thread t2(write_thread, std::ref(g));
    t1.join();
    t2.join();
  }
}
```
在读写开始之前，各个 shared_ptr 的状态为：
![读写开始前的状态](http://www.cppblog.com/images/cppblog_com/Solstice/WindowsLiveWriter/shared_ptr_1398A/sp5_07e916af-46fb-42bc-aa87-14fc89701e54.png)
接着线程 A 执行 x=g, 此时完成了步骤 1 ，但是没来得及执行步骤 2 ，然后切换到了写线程，此时的状态为：
![读线程只完成步骤1](http://www.cppblog.com/images/cppblog_com/Solstice/WindowsLiveWriter/shared_ptr_1398A/sp6_f541f13e-cae2-4b11-99e0-8e025806e30c.png)
接着写线程执行 g=n; 两个步骤一起完成了
先是步骤 1 完成，此时状态为：
![写线程完成步骤1](http://www.cppblog.com/images/cppblog_com/Solstice/WindowsLiveWriter/shared_ptr_1398A/sp7_028b8bdf-16c7-4f2e-b05b-b31ae146bcd3.png)
再是步骤 2 完成，此时状态为：
![写线程完成步骤2](http://www.cppblog.com/images/cppblog_com/Solstice/WindowsLiveWriter/shared_ptr_1398A/sp8_9f43d6ad-4455-47da-8c23-e72461427c98.png)
此时 Foo1 对象已经销毁，x.ptr 成了**空悬指针**，再**调用其接口函数会 core dump**
最后回到写线程，完成步骤 2 ，此时状态为：
![读线程完成步骤2](http://www.cppblog.com/images/cppblog_com/Solstice/WindowsLiveWriter/shared_ptr_1398A/sp9_9d6b5777-f9f4-483d-b1ba-99ae0086ffaf.png)

多线程无保护地读写 g，造成了“x 是空悬指针”的后果。这正是多线程读写同一个 shared_ptr 必须加锁的原因。
当然，race condition 远不止这一种，其他线程交织（interweaving）有可能会造成其他错误。

对上面的 test.cpp 进行测试，可以发现典型的 2 种错误：
```shell
double free or corruption (fasttop)
Aborted

Segmentation fault
```
double free 是因为当写进程只完成步骤 1 就开始执行读的两个步骤时， x 的指针和 ref_count 不是指向同一个对象！


参考文档：
[为什么多线程读写 shared_ptr 要加锁？](http://www.cppblog.com/Solstice/archive/2013/01/28/197597.html)
[shared_ptr_thread_safety](https://www.boost.org/doc/libs/1_73_0/libs/smart_ptr/doc/html/smart_ptr.html#shared_ptr_thread_safety)